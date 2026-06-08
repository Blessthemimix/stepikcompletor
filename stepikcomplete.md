// ==UserScript==
// @name         Stepik Auto-Completer AI+ (OpenRouter Edition)
// @namespace    http://tampermonkey.net/
// @version      3.0
// @description  Автоответчик для Stepik с поддержкой бесплатного Gemini через OpenRouter
// @match        https://stepik.org/*
// @grant        GM_xmlhttpRequest
// @run-at       document-idle
// ==/UserScript==

(function () {
    'use strict';

    // ─── НАСТРОЙКИ ИИ (Сюда вставляешь ключ от OpenRouter) ─────────────────────
    const AI_API_KEY = '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';
    const AI_API_URL = 'https://openrouter.ai/api/v1/chat/completions';
    const AI_MODEL   = 'google/gemma-4-31b-it';// Полностью бесплатная модель Gemini
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
        const m = href.match(/stepik\.org\/lesson\/(?:[^/]+--)?(\d+)\/step\/(\d+)/);
        if (!m) return null;
        return { lessonId: m[1], stepPos: parseInt(m[2]) };
    }

    async function getStep(lessonId, stepPos) {
        const data = await api('GET', 'steps?lesson=' + lessonId);
        const steps = data.steps || [];
        return steps[stepPos - 1] || null;
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
        const data = await api('POST', 'submissions', {
            submission: { attempt: attemptId, reply },
        });
        const sub = (data.submissions || [])[0];
        if (!sub) return 'error';

        for (let i = 0; i < 20; i++) {
            await sleep(700);
            try {
                const chk = await api('GET', 'submissions/' + sub.id);
                const s = (chk.submissions || [])[0];
                if (s && s.status !== 'evaluation') return s.status;
            } catch { /* retry */ }
        }
        return 'timeout';
    }

    // ─── Интеграция с OpenRouter через GM_xmlhttpRequest ───────────────────────
    function askAI(stepText, type, attempt) {
        if (!AI_API_KEY || AI_API_KEY.includes('ВСТАВЬ_СЮДА_')) {
            return Promise.reject(new Error('Не настроен AI_API_KEY в коде скрипта.'));
        }

        const systemPrompt = `Ты — автоматический решатель задач для платформы Stepik.
Анализируй текст задания (включая HTML теги и код) и возвращай ответ строго в формате JSON.
Твой ответ должен содержать ОДНО поле "ans". Никакого лишнего текста, объяснений или разметки markdown вокруг JSON.

Форматы поля "ans" в зависимости от типа задания:
- Если тип "number": в поле "ans" должна быть строка с чистым числом (например: "11" или "3.14").
- Если тип "string": строка-ответ.
- Если тип "math": математическая формула в формате Stepik/LaTeX (например: "x**2 + y").
- Если тип "code": готовый, рабочий код на выбранном языке программирования, решающий задачу по входным данным.`;

        const userPrompt = `Тип задания: ${type}\nТекст задания:\n${stepText}`;

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
                    temperature: 0.1
                }),
                onload: function(response) {
                    if (response.status !== 200) {
                        reject(new Error(`Ошибка ИИ API: ${response.status} - ${response.responseText}`));
                        return;
                    }
                    try {
                        let aiText = JSON.parse(response.responseText).choices[0].message.content.trim();
                        // Очистка от markdown-оберток
                        aiText = aiText.replace(/^```json\s*/i, '').replace(/```$/, '').trim();
                        const parsed = JSON.parse(aiText);
                        const answer = parsed.ans;

                        if (type === 'number') resolve({ number: String(answer).trim() });
                        else if (type === 'string') resolve({ string: String(answer) });
                        else if (type === 'math') resolve({ formula: String(answer).trim() });
                        else if (type === 'code') {
                            let lang = 'python3';
                            if (attempt && attempt.dataset && attempt.dataset.code_templates) {
                                const templates = Object.keys(attempt.dataset.code_templates);
                                if (templates.length > 0) lang = templates[0];
                            }
                            resolve({ code: answer, language: lang });
                        } else {
                            resolve(null);
                        }
                    } catch (err) {
                        reject(new Error('Не удалось распарсить ответ ИИ.'));
                    }
                },
                onerror: function(err) {
                    reject(new Error('Сетевая ошибка при запросе к OpenRouter.'));
                }
            });
        });
    }

    // ─── Перебор тестов (Choice) ──────────────────────────────────────────────
    async function solveChoice(step, attempt) {
        const stepId = step.id;
        const isMultiple = !!(attempt.dataset && attempt.dataset.is_multiple_choice);
        const options = (attempt.dataset && attempt.dataset.options) || [];
        const n = options.length;
        if (n === 0) { setStatus('❌ Варианты ответов не найдены.'); return false; }

        if (!isMultiple) {
            setStatus(`Одиночный выбор: тестирую ${n} вариантов…`);
            for (let i = 0; i < n; i++) {
                if (!running) return false;
                const choices = Array(n).fill(false);
                choices[i] = true;
                setStatus(`Проверяю вариант ${i + 1}/${n}…`);
                const status = await submitAndWait(attempt.id, { choices });
                if (status === 'correct') { setStatus('✅ Успешно решено!'); return true; }
                if (i < n - 1) {
                    await sleep(500);
                    attempt = await createAttempt(stepId);
                    if (!attempt) break;
                }
            }
        } else {
            setStatus('Множественный выбор: проверяю базовые комбинации…');
            let status = await submitAndWait(attempt.id, { choices: Array(n).fill(true) });
            if (status === 'correct') { setStatus('✅ Успешно решено (все варианты)!'); return true; }

            for (let i = 0; i < n; i++) {
                if (!running) return false;
                await sleep(500);
                attempt = await createAttempt(stepId);
                if (!attempt) break;
                const choices = Array(n).fill(false);
                choices[i] = true;
                setStatus(`Мульти-выбор: одиночный вариант ${i + 1}/${n}…`);
                status = await submitAndWait(attempt.id, { choices });
                if (status === 'correct') { setStatus('✅ Успешно решено!'); return true; }
            }
        }
        return false;
    }

    function navigateNext(lessonId, stepPos) {
        const links = document.querySelectorAll('a[href*="/step/"]');
        for (const link of links) {
            const m = link.href.match(/\/step\/(\d+)/);
            if (m && parseInt(m[1]) === stepPos + 1) {
                link.click();
                return;
            }
        }
        const base = location.href.replace(/\/step\/\d+.*/, '');
        location.href = base + '/step/' + (stepPos + 1);
    }

    async function processStep() {
        if (processing) return;
        processing = true;

        try {
            const info = parseLessonUrl();
            if (!info) { setStatus('Шаг лекции не обнаружен.'); return; }

            setStatus('Загружаю инфо о шаге…');
            const step = await getStep(info.lessonId, info.stepPos);

            if (!step) {
                setStatus('🎉 Последний шаг в уроке выполнен!');
                running = false;
                updateBtn();
                return;
            }

            if (await isAlreadySolved(step.id)) {
                setStatus('Уже решено — иду дальше…');
                await sleep(600);
                navigateNext(info.lessonId, info.stepPos);
                return;
            }

            const type = (step.block && step.block.name) || 'unknown';
            setStatus('Тип шага: ' + type);

            if (['video', 'text', 'animation', 'table', 'image'].includes(type)) {
                setStatus('Пропускаю теорию/видео…');
                await sleep(1000);
                if (running) navigateNext(info.lessonId, info.stepPos);
                return;
            }

            let attempt = await createAttempt(step.id);
            if (!attempt) { setStatus('❌ Не удалось создать попытку решения.'); return; }

            if (type === 'choice') {
                const solved = await solveChoice(step, attempt);
                if (!solved) {
                    setStatus('⏸ Тест не подобрался. Пауза.');
                    running = false;
                    updateBtn();
                    return;
                }
            }
            else if (['number', 'string', 'math', 'code'].includes(type)) {
                setStatus(`🤖 ИИ думает над задачей типа "${type}"…`);
                try {
                    const replyPayload = await askAI(step.block.text, type, attempt);
                    if (replyPayload) {
                        setStatus('🤖 ИИ сформировал ответ. Отправляю…');
                        const status = await submitAndWait(attempt.id, replyPayload);
                        if (status === 'correct') {
                            setStatus('✅ ИИ ответил правильно!');
                        } else {
                            setStatus(`❌ ИИ ошибся (Статус: ${status}). Пауза.`);
                            running = false;
                            updateBtn();
                            return;
                        }
                    }
                } catch (aiErr) {
                    setStatus(`⏸ Ошибка ИИ: ${aiErr.message}. Нужен ручной ввод.`);
                    running = false;
                    updateBtn();
                    return;
                }
            } else {
                setStatus(`⏸ Тип "${type}" не поддерживается. Стоп.`);
                running = false;
                updateBtn();
                return;
            }

            if (running) {
                await sleep(1200);
                navigateNext(info.lessonId, info.stepPos);
            }
        } catch (e) {
            setStatus('Ошибка: ' + e.message);
            console.error('[Stepik Auto]', e);
        } finally {
            processing = false;
        }
    }

    setInterval(async () => {
        if (location.href !== lastUrl) {
            lastUrl = location.href;
            if (running && parseLessonUrl()) {
                await sleep(2500);
                await processStep();
            }
        }
    }, 500);

    function setStatus(msg) {
        const el = document.getElementById('_sa_status');
        if (el) el.textContent = msg;
        console.log('[Stepik Auto]', msg);
    }

    function updateBtn() {
        const btn = document.getElementById('_sa_btn');
        if (!btn) return;
        btn.textContent = running ? '⏸ Стоп' : '▶ Старт';
        btn.style.background = running ? '#dc2626' : '#6366f1';
    }

    function buildPanel() {
        if (document.getElementById('_sa_panel')) return;

        const panel = document.createElement('div');
        panel.id = '_sa_panel';
        panel.style.cssText = [
            'position:fixed', 'bottom:22px', 'right:22px', 'z-index:2147483647',
            'background:#0d0d18', 'color:#e2e8f0', 'padding:16px 18px',
            'border-radius:14px', 'font-family:system-ui,sans-serif',
            'font-size:13px', 'min-width:220px', 'max-width:260px',
            'box-shadow:0 8px 40px rgba(0,0,0,0.6)',
            'border:1px solid rgba(255,255,255,0.07)',
        ].join(';');

        panel.innerHTML = `
            <div style="display:flex;align-items:center;gap:8px;margin-bottom:12px;">
                <span style="font-size:18px;">⚡</span>
                <span style="font-weight:700;font-size:14px;letter-spacing:.3px;">Stepik Auto AI+</span>
            </div>
            <div id="_sa_status" style="
                color:#94a3b8;margin-bottom:13px;font-size:11.5px;
                line-height:1.5;min-height:34px;
            ">Вставьте OpenRouter ключ, обновите страницу и нажмите Старт.</div>
            <button id="_sa_btn" style="
                width:100%;padding:9px 12px;background:#6366f1;color:#fff;
                border:none;border-radius:9px;cursor:pointer;
                font-size:13px;font-weight:600;letter-spacing:.3px;
                transition:background .2s;
            ">▶ Старт</button>
            <div style="margin-top:11px;padding-top:10px;border-top:1px solid rgba(255,255,255,.06);
                        color:#475569;font-size:10.5px;line-height:1.6;">
                ✓ Тесты (Брутфорс)<br>
                ✓ Числа / Математика (Через ИИ)<br>
                ✓ Написание кода (Через ИИ)
            </div>
        `;

        document.body.appendChild(panel);

        document.getElementById('_sa_btn').addEventListener('click', async () => {
            running = !running;
            updateBtn();
            if (running) {
                if (!parseLessonUrl()) {
                    setStatus('⚠️ Зайдите в шаг урока!');
                    running = false;
                    updateBtn();
                    return;
                }
                await processStep();
            } else {
                setStatus('Остановлено.');
            }
        });
    }

    function init() { buildPanel(); }

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', () => setTimeout(init, 1500));
    } else {
        setTimeout(init, 1500);
    }
})();
