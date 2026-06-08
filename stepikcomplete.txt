// ==UserScript==
// @name         Stepik Auto-Completer AI+ (v4.9.2 Smart Flow Real Fix)
// @namespace    http://tampermonkey.net/
// @version      4.9.2
// @description  Автоответчик для Stepik с мягкой навигацией, системой повторных попыток ИИ и автопереходом по разделам
// @match        *://*.stepik.org/*
// @match        *://stepik.org/*
// @grant        GM_xmlhttpRequest
// @run-at       document-start
// ==/UserScript==

(function () {
    'use strict';

    console.log("%c[Stepik AI] СКРИПТ УСПЕШНО ИНИЦИАЛИЗИРОВАН (v4.9.2 Тотальный фикс)!", "color: #10b981; font-size: 14px; font-weight: bold; background: #0f172a; padding: 4px 8px; border-radius: 4px;");

    // ─── НАСТРОЙКИ ИИ ────────────────────────────────────────────────────────
    const AI_API_KEY = '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';
    const AI_API_URL = 'https://openrouter.ai/api/v1/chat/completions';
    const AI_MODEL   = 'google/gemma-4-31b-it';
    // ──────────────────────────────────────────────────────────────────────────

    let running = false;
    let processing = false;
    let lastUrl = location.href;

    const sleep = ms => new Promise(r => setTimeout(r, ms));

    function getCookie(name) {
        const m = document.cookie.match(new RegExp(`(?:^|; )${name}=([^;]*)`));
        return m ? decodeURIComponent(m[1]) : '';
    }

    async function api(method, path, body) {
        const resp = await fetch('https://stepik.org/api/' + path, {
            method,
            credentials: 'include',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRFToken': getCookie('csrftoken'),
                'Referer': 'https://stepik.org',
            },
            body: body !== undefined ? JSON.stringify(body) : undefined,
        });
        if (!resp.ok) throw new Error('HTTP ' + resp.status);
        return resp.json();
    }

    function parseLessonUrl(href) {
        href = href || location.href;
        const m = href.match(/\/lesson\/(?:[^\/]+-)?(\d+)\/step\/(\d+)/);
        if (!m) return null;
        return { lessonId: m[1], stepPos: parseInt(m[2]) };
    }

    async function getStep(lessonId, stepPos) {
        console.log(`[Stepik AI] Запрос данных шага №${stepPos}...`);
        const lessonData = await api('GET', 'lessons/' + lessonId);
        const lesson = (lessonData.lessons || [])[0];
        if (!lesson || !lesson.steps || stepPos > lesson.steps.length) return null;

        const stepId = lesson.steps[stepPos - 1];
        const stepData = await api('GET', 'steps/' + stepId);
        return (stepData.steps || [])[0] || null;
    }

    async function isAlreadySolved(stepId) {
        try {
            const data = await api('GET', 'submissions?step=' + stepId + '&order=desc&page=1');
            return (data.submissions || []).some(s => s.status === 'correct');
        } catch { return false; }
    }

    async function createAttempt(stepId) {
        const data = await api('POST', 'attempts', { attempt: { step: stepId } });
        return (data.attempts || [])[0] || null;
    }

    async function submitAndWait(attemptId, reply) {
        console.log('[Stepik AI] Отправка решения в API Stepik...');
        const data = await api('POST', 'submissions', {
            submission: { attempt: attemptId, reply },
        });
        const sub = (data.submissions || [])[0];
        if (!sub) {
            console.error('[Stepik AI] Ошибка: Не удалось создать отправку в API.');
            return 'error';
        }

        console.log(`[Stepik AI] Отправка успешно создана (ID: ${sub.id}). Ожидаем вердикт грайдера...`);
        for (let i = 0; i < 20; i++) {
            await sleep(1000);
            try {
                const chk = await api('GET', 'submissions/' + sub.id);
                const s = (chk.submissions || [])[0];
                if (s && s.status !== 'evaluation') {
                    console.log(`[Stepik AI] Грайдер завершил проверку. Результат сервера: %c${s.status.toUpperCase()}`, s.status === 'correct' ? 'color: #10b981; font-weight: bold;' : 'color: #ef4444; font-weight: bold;');
                    return s.status;
                }
            } catch { /* retry */ }
        }
        return 'timeout';
    }

    function askAI(stepText, type, attempt, isRetry = false) {
        if (!AI_API_KEY || AI_API_KEY.includes('ВСТАВЬ_СЮДА_')) {
            return Promise.reject(new Error('Ключ OpenRouter не найден в коде скрипта.'));
        }

        let systemPrompt = `Ты — автоматический решатель задач для Stepik. Отвечай строго в формате JSON с единственным полем "ans". Без markdown разметки (никаких \`\`\`json).`;

        if (type === 'code') {
            systemPrompt += ` Напиши готовую программу на Python 3. Она должна читать данные из input() и выводить результат в stdout (print()). Помести весь текст кода в строковое поле "ans", используя \\n для переноса строк.`;
        }
        else if (type === 'string' || type === 'number') {
            systemPrompt += ` Ты должен дать точный лаконичный ответ на задание. Если в тексте содержится вопрос "Что покажет приведённый ниже код?", выполни его как Python-интерпретатор и запиши в поле "ans" ИСКЛЮЧИТЕЛЬНО то, что выведется в консоль. Никаких лишних слов, объяснений или вводных фраз!`;
        }

        let userPrompt = `Тип задания: ${type}\nТекст задания:\n${stepText}`;

        if (isRetry) {
            userPrompt += `\n\n⚠️ ВНИМАНИЕ: Твой предыдущий вариант ответа система посчитала НЕВЕРНЫМ. Пожалуйста, внимательно перечитай условие задачи, найди ошибку в своей логике и выдай ДРУГОЙ, абсолютно точный и правильный ответ.`;
        }

        return new Promise((resolve, reject) => {
            GM_xmlhttpRequest({
                method: 'POST',
                url: AI_API_URL,
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${AI_API_KEY}`
                },
                data: JSON.stringify({
                    model: AI_MODEL,
                    messages: [
                        { role: 'system', content: systemPrompt },
                        { role: 'user', content: userPrompt }
                    ],
                    temperature: isRetry ? 0.4 : 0.1
                }),
                onload: function(response) {
                    if (response.status !== 200) {
                        reject(new Error(`API Error ${response.status}`));
                        return;
                    }
                    try {
                        let rawContent = JSON.parse(response.responseText).choices[0].message.content.trim();
                        rawContent = rawContent.replace(/^```json\s*/i, '').replace(/```$/, '').trim();

                        let answer = null;
                        try {
                            const parsed = JSON.parse(rawContent);
                            answer = parsed.ans;
                        } catch (jsonErr) {
                            const match = rawContent.match(/"ans"\s*:\s*"([\s\S]*?)"\s*}/);
                            if (match) {
                                answer = match[1].replace(/\\n/g, '\n').replace(/\\"/g, '"').replace(/\\t/g, '\t');
                            } else { throw jsonErr; }
                        }

                        if (typeof answer === 'string') {
                            answer = answer.replace(/^(ответ|результат|output|answer)\s*:\s*/i, '').trim();
                            if (answer.startsWith('"') && answer.endsWith('"')) answer = answer.slice(1, -1);
                            if (answer.startsWith("'") && answer.endsWith("'")) answer = answer.slice(1, -1);
                            answer = answer.trim();
                        }

                        console.log(`[Stepik AI] Очищенный ответ для отправки:`, answer);

                        if (type === 'number') {
                            const numMatch = String(answer).match(/-?\d+[\.,]?\d*/);
                            const finalNum = numMatch ? numMatch[0].replace(',', '.') : String(answer).trim();
                            resolve({ number: finalNum });
                        }
                        else if (type === 'string') resolve({ string: String(answer) });
                        else if (type === 'math') resolve({ formula: String(answer).trim() });
                        else if (type === 'code') {
                            let lang = 'python3';
                            if (attempt && attempt.dataset && attempt.dataset.code_templates) {
                                const templates = Object.keys(attempt.dataset.code_templates);
                                if (templates.length > 0) lang = templates[0];
                            }
                            resolve({ code: answer, language: lang });
                        } else resolve(null);
                    } catch (err) { reject(new Error('Ошибка разбора JSON от ИИ.')); }
                },
                onerror: function() { reject(new Error('Сетевая ошибка OpenRouter.')); }
            });
        });
    }

    async function solveChoice(step, attempt) {
        const stepId = step.id;
        const isMultiple = !!(attempt.dataset && attempt.dataset.is_multiple_choice);
        const options = (attempt.dataset && attempt.dataset.options) || [];
        const n = options.length;
        if (n === 0) return false;

        if (!isMultiple) {
            for (let i = 0; i < n; i++) {
                if (!running) return false;
                const choices = Array(n).fill(false);
                choices[i] = true;
                setStatus(`Вариант ${i + 1}/${n}…`);
                const status = await submitAndWait(attempt.id, { choices });
                if (status === 'correct') return true;
                if (i < n - 1) {
                    await sleep(500);
                    attempt = await createAttempt(stepId);
                    if (!attempt) break;
                }
            }
        } else {
            let status = await submitAndWait(attempt.id, { choices: Array(n).fill(true) });
            if (status === 'correct') return true;

            for (let i = 0; i < n; i++) {
                if (!running) return false;
                await sleep(500);
                attempt = await createAttempt(stepId);
                if (!attempt) break;
                const choices = Array(n).fill(false);
                choices[i] = true;
                setStatus(`Мульти-тест ${i + 1}/${n}…`);
                status = await submitAndWait(attempt.id, { choices });
                if (status === 'correct') return true;
            }
        }
        return false;
    }

    function clickNextDOM() {
        const nextBtn = document.querySelector('.lesson-pagination__next, .next-button, button[class*="next"], a[class*="next"]');
        if (nextBtn) {
            nextBtn.click();
            return true;
        }

        const allElements = document.querySelectorAll('button, a');
        for (let el of allElements) {
            if (el.textContent && (el.textContent.includes('Следующий шаг') || el.textContent.includes('Вперед') || el.textContent.includes('Вперёд'))) {
                el.click();
                return true;
            }
        }

        const activeTab = document.querySelector('.step-navigation__item_active, [class*="step-navigation__item--active"]');
        if (activeTab && activeTab.nextElementSibling) {
            const link = activeTab.nextElementSibling.querySelector('a, button') || activeTab.nextElementSibling;
            if (link) {
                link.click();
                return true;
            }
        }
        return false;
    }

    async function navigateNext(lessonId, stepPos) {
        console.log(`[Stepik AI] Переходим к следующему шагу...`);
        const success = clickNextDOM();

        if (!success) {
            console.log(`[Stepik AI] Мягкий переход не удался, применяем резервный хард-релоад.`);
            const base = location.href.replace(/\/step\/\d+.*/, '');
            const nextUrl = base + '/step/' + (stepPos + 1);
            location.href = nextUrl;
        }
    }

    async function processStep() {
        if (processing) return;
        processing = true;

        try {
            const info = parseLessonUrl();
            if (!info) { setStatus('Страница теории / не шаг.'); return; }

            setStatus('Анализ шага…');
            const step = await getStep(info.lessonId, info.stepPos);

            if (!step) {
                setStatus('Конец урока. Перехожу в следующий раздел…');
                await sleep(1500);
                const moved = clickNextDOM();
                if (!moved) {
                    setStatus('Все уроки завершены!');
                    running = false;
                    updateBtn();
                }
                return;
            }

            if (await isAlreadySolved(step.id)) {
                setStatus('Решено. Пропуск…');
                await sleep(500);
                navigateNext(info.lessonId, info.stepPos);
                return;
            }

            const type = (step.block && step.block.name) || 'unknown';
            setStatus('Тип: ' + type);

            if (['video', 'text', 'animation', 'table', 'image'].includes(type)) {
                await sleep(500);
                if (running) navigateNext(info.lessonId, info.stepPos);
                return;
            }

            let attempt = await createAttempt(step.id);
            if (!attempt) { setStatus('Ошибка попытки.'); return; }

            if (type === 'choice') {
                const solved = await solveChoice(step, attempt);
                if (!solved) {
                    setStatus(`⏭️ Тест не подобран. Скип…`);
                    await sleep(1500);
                }
            }
            else if (['number', 'string', 'math', 'code'].includes(type)) {
                setStatus(`🤖 Решаю [${type}], Попытка 1…`);
                try {
                    let replyPayload = await askAI(step.block.text, type, attempt, false);
                    if (replyPayload) {
                        let status = await submitAndWait(attempt.id, replyPayload);

                        if (status !== 'correct' && running) {
                            console.log(`[Stepik AI] Попытка 1 неверна. Запрашиваю ИИ повторно с работой над ошибками...`);
                            setStatus(`❌ Неверно. Попытка 2 (ИИ исправляется)…`);
                            await sleep(1000);

                            attempt = await createAttempt(step.id);
                            if (attempt) {
                                replyPayload = await askAI(step.block.text, type, attempt, true);
                                if (replyPayload) {
                                    status = await submitAndWait(attempt.id, replyPayload);
                                }
                            }
                        }

                        if (status !== 'correct') {
                            console.log(`[Stepik AI] Попытка 2 тоже неверна. Пропускаем задачу (Скип).`);
                            setStatus(`⏭️ Неверно 2 раза. Скип задачи…`);
                            await sleep(1500);
                        }
                    }
                } catch (aiErr) {
                    setStatus(`Ошибка: ${aiErr.message}`);
                    running = false;
                    updateBtn();
                    return;
                }
            }

            if (running) {
                await sleep(1000);
                navigateNext(info.lessonId, info.stepPos);
            }
        } catch (e) {
            setStatus('Сбой: ' + e.message);
            running = false;
            updateBtn();
        } finally { // Вот ТЕПЕРЬ здесь написано finally как положено!
            processing = false;
        }
    }

    function setStatus(msg) {
        const el = document.getElementById('_sa_status');
        if (el) el.textContent = msg;
    }

    function updateBtn() {
        const btn = document.getElementById('_sa_btn');
        if (btn) {
            btn.textContent = running ? '⏸ Стоп' : '▶ Старт';
            btn.style.background = running ? '#dc2626' : '#6366f1';
        }
    }

    function buildPanel() {
        if (document.getElementById('_sa_panel')) return;

        const panel = document.createElement('div');
        panel.id = '_sa_panel';
        panel.style.cssText = 'position:fixed !important; bottom:22px !important; right:22px !important; z-index:2147483647 !important; background:#0d0d18 !important; color:#e2e8f0 !important; padding:16px 18px !important; border-radius:14px !important; font-family:system-ui,sans-serif !important; font-size:13px !important; min-width:220px !important; box-shadow:0 8px 40px rgba(0,0,0,0.6) !important; border:1px solid rgba(255,255,255,0.07) !important; display:block !important; text-align:left !important;';
        panel.innerHTML = `<div style="display:flex;align-items:center;gap:8px;margin-bottom:12px;"><span style="font-size:18px;">⚡</span><span style="font-weight:700;font-size:14px;color:#fff;">Stepik Flow v4.9.2</span></div><div id="_sa_status" style="color:#94a3b8;margin-bottom:13px;font-size:11.5px;min-height:34px;">Скрипт ожидает команду...</div><button id="_sa_btn" style="width:100%;padding:9px 12px;background:#6366f1;color:#fff;border:none;border-radius:99px;cursor:pointer;font-size:13px;font-weight:600;display:block !important;visibility:visible !important;">▶ Старт</button>`;

        const root = document.body || document.documentElement;
        if (root) { root.appendChild(panel); }

        document.getElementById('_sa_btn').addEventListener('click', async () => {
            running = !running;
            updateBtn();
            if (running) { await processStep(); } else { setStatus('Остановлено.'); }
        });
        updateBtn();
    }

    setInterval(() => {
        if (!document.getElementById('_sa_panel')) { buildPanel(); }

        if (location.href !== lastUrl) {
            lastUrl = location.href;
            if (running && parseLessonUrl()) {
                setTimeout(processStep, 1500);
            }
        }
    }, 400);

})();
