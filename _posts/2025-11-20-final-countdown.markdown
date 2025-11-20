---
layout: default
title: "Final Countdown!"
date: 2025-11-20 00:00:00 +0000
---

<style>
    .timer-box {
        background: white;
        width: 70%;
        margin: 20px auto;
        padding: 20px;
        border-radius: 10px;
        box-shadow: 0 0 10px #aaa;
    }

    .timer-title {
        font-size: 1.6rem;
        margin-bottom: 10px;
        font-weight: bold;
        color: #333;
    }

    .progress-container {
        width: 100%;
        height: 25px;
        background: #ddd;
        border-radius: 10px;
        overflow: hidden;
        margin-top: 10px;
    }

    .progress-bar {
        height: 100%;
        width: 0%;
        background: #4caf50;
        transition: width 2s ease;
    }
</style>

<div id="countdownList"></div>

<script>
    function addCountdown(titleValue, startValue, endValue) {
        if (!startValue || !endValue) return;

        const startDate = new Date(startValue);
        const endDate = new Date(endValue);
        const totalTime = endDate - startDate;
        const timerId = "timer-" + Math.random().toString(36).substr(2, 9);

        const box = document.createElement("div");
        box.classList.add("timer-box");
        box.innerHTML = `
        <div class="timer-title">${titleValue}</div>
        <p>Start Date: ${startDate.toDateString()}</p>
        <p>End Date: ${endDate.toDateString()}</p>
        <div id="${timerId}"></div>
        <div class="progress-container">
            <div class="progress-bar" id="progress-${timerId}"></div>
        </div>
        <p id="progress-text-${timerId}"></p>
    `;

        document.getElementById("countdownList").appendChild(box);

        setInterval(() => updateCountdown(startDate, endDate, totalTime, timerId), 1000);
        setTimeout(() => updateCountdown(startDate, endDate, totalTime, timerId, true), 200);
    }

    function updateCountdown(startDate, endDate, totalTime, timerId, isInitial = false) {
        const now = new Date();
        const timeLeft = endDate - now;
        const timerEl = document.getElementById(timerId);

        if (timeLeft <= 0) {
            timerEl.innerHTML = "TIME IS UP!";
            document.getElementById(`progress-${timerId}`).style.width = "100%";
            document.getElementById(`progress-text-${timerId}`).innerHTML = "100% Completed";
            return;
        }

        const days = Math.floor(timeLeft / (1000 * 60 * 60 * 24));
        const hours = Math.floor((timeLeft / (1000 * 60 * 60)) % 24);
        const minutes = Math.floor((timeLeft / (1000 * 60)) % 60);
        const seconds = Math.floor((timeLeft / 1000) % 60);
        timerEl.innerHTML = `${days} Days ${hours} Hours ${minutes} Minutes ${seconds} Seconds`;

        const timePassed = now - startDate;
        let progressPercent = (timePassed / totalTime) * 100;
        progressPercent = Math.min(progressPercent, 100);

        const progressBar = document.getElementById(`progress-${timerId}`);

        if (isInitial) {
            progressBar.style.transition = "none";
            progressBar.style.width = "0%";
            setTimeout(() => {
                progressBar.style.transition = "width 2s ease";
                progressBar.style.width = progressPercent + "%";
            }, 100);
        } else {
            progressBar.style.width = progressPercent + "%";
        }

        document.getElementById(`progress-text-${timerId}`).innerHTML = progressPercent.toFixed(2) + "% Completed";
    }

    // EXAMPLE COUNTDOWNS (Auto Loaded)
    addCountdown("Stamp 4", "2025-11-10T00:00:00", "2028-11-10T23:59:59");
    addCountdown("Citizenship", "2025-02-10T00:00:00", "2030-02-10T23:59:59");
</script>
