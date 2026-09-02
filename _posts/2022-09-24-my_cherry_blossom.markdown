---
title: "My Cherry Blossom"
date: 2022-09-24 00:00:00 +0000
permalink: /my-cherry-blossom
layout: null
hidden: true
---

<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />

  <title>Our Time Together ❤️</title>

  <style>
    * {
      box-sizing: border-box;
      margin: 0;
      padding: 0;
    }

    body {
      min-height: 100vh;
      overflow: hidden;
      font-family: Arial, Helvetica, sans-serif;
      background:
        radial-gradient(circle at top, #4a1730 0%, #1d0b18 45%, #090308 100%);
      color: white;
    }

    .hearts {
      position: fixed;
      inset: 0;
      z-index: 0;
      pointer-events: none;
      overflow: hidden;
    }

    .heart {
      position: absolute;
      color: #ff4f86;
      opacity: 0.22;
      user-select: none;
      animation: float 8s ease-in-out infinite alternate;
    }

    @keyframes float {
      from {
        transform: translateY(0) rotate(-8deg) scale(1);
      }

      to {
        transform: translateY(-25px) rotate(8deg) scale(1.15);
      }
    }

    .container {
      position: relative;
      z-index: 10;
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
      padding: 25px;
    }

    .content {
      width: 100%;
      max-width: 950px;
      text-align: center;
    }

    .small-title {
      font-size: 15px;
      letter-spacing: 4px;
      text-transform: uppercase;
      opacity: 0.65;
      margin-bottom: 15px;
    }

    h1 {
      font-size: clamp(35px, 7vw, 75px);
      margin-bottom: 15px;
    }

    .main-heart {
      color: #ff4f86;
      display: inline-block;
      animation: heartbeat 1.2s infinite;
    }

    @keyframes heartbeat {
      0%, 100% {
        transform: scale(1);
      }

      15% {
        transform: scale(1.18);
      }

      30% {
        transform: scale(1);
      }

      45% {
        transform: scale(1.1);
      }

      60% {
        transform: scale(1);
      }
    }

    .subtitle {
      opacity: 0.7;
      margin-bottom: 45px;
      font-size: 17px;
    }

    .timer {
      display: grid;
      grid-template-columns: repeat(5, 1fr);
      gap: 15px;
      margin-bottom: 35px;
    }

    .time-box {
      padding: 25px 10px;
      border-radius: 20px;
      background: rgba(255, 255, 255, 0.07);
      border: 1px solid rgba(255, 255, 255, 0.12);
      backdrop-filter: blur(10px);
      -webkit-backdrop-filter: blur(10px);
    }

    .number {
      display: block;
      font-size: clamp(30px, 5vw, 55px);
      font-weight: bold;
      color: #ff7ca3;
      margin-bottom: 8px;
    }

    .label {
      font-size: 13px;
      opacity: 0.6;
      text-transform: uppercase;
      letter-spacing: 2px;
    }

    .message {
      font-size: 18px;
      line-height: 1.7;
      opacity: 0.85;
    }

    .heart-count {
      margin-top: 12px;
      font-size: 15px;
      opacity: 0.6;
    }

    .heart-count strong {
      color: #ff7ca3;
    }

    @media (max-width: 700px) {
      body {
        overflow: auto;
      }

      .timer {
        grid-template-columns: repeat(2, 1fr);
      }

      .time-box:last-child {
        grid-column: 1 / -1;
      }

      .subtitle {
        margin-bottom: 25px;
      }
    }
  </style>
</head>

<body>

  <div class="hearts" id="hearts"></div>

  <main class="container">
    <div class="content">

      <div class="small-title">
        Every moment together
      </div>

      <h1>
        Our Story
        <span class="main-heart">♥</span>
      </h1>

      <p class="subtitle" id="startText"></p>

      <div class="timer">

        <div class="time-box">
          <span class="number" id="years">0</span>
          <span class="label">Years</span>
        </div>

        <div class="time-box">
          <span class="number" id="days">0</span>
          <span class="label">Days</span>
        </div>

        <div class="time-box">
          <span class="number" id="hours">0</span>
          <span class="label">Hours</span>
        </div>

        <div class="time-box">
          <span class="number" id="minutes">0</span>
          <span class="label">Minutes</span>
        </div>

        <div class="time-box">
          <span class="number" id="seconds">0</span>
          <span class="label">Seconds</span>
        </div>

      </div>

      <p class="message">
        One heart for every day we have shared. ❤️
      </p>

      <div class="heart-count">
        So far, we have collected
        <strong id="totalDays">0</strong>
        hearts.
      </div>

    </div>
  </main>

  <script>
    /*
      ============================================
      CHANGE THE START DATE HERE
      ============================================

      Format:
      YYYY-MM-DDTHH:MM:SS

      Example:
      2024-06-15T20:30:00
    */

    const START_DATE = "2022-09-24T00:00:00";

    const startDate = new Date(START_DATE);

    const heartsContainer = document.getElementById("hearts");

    const yearsElement = document.getElementById("years");
    const daysElement = document.getElementById("days");
    const hoursElement = document.getElementById("hours");
    const minutesElement = document.getElementById("minutes");
    const secondsElement = document.getElementById("seconds");

    const totalDaysElement = document.getElementById("totalDays");
    const startTextElement = document.getElementById("startText");

    /*
      Display the start date in a readable format
    */
    startTextElement.textContent =
      "Since " +
      startDate.toLocaleString("en-US", {
        day: "numeric",
        month: "long",
        year: "numeric",
        hour: "2-digit",
        minute: "2-digit"
      });

    let visibleHeartCount = 0;

    /*
      Create one heart
    */
    function createHeart() {
      const heart = document.createElement("span");

      heart.className = "heart";
      heart.textContent = "♥";

      heart.style.left = Math.random() * 100 + "%";
      heart.style.top = Math.random() * 100 + "%";

      const size = 8 + Math.random() * 22;
      heart.style.fontSize = size + "px";

      heart.style.opacity = 0.08 + Math.random() * 0.25;

      heart.style.animationDuration =
        5 + Math.random() * 8 + "s";

      heart.style.animationDelay =
        -Math.random() * 8 + "s";

      heartsContainer.appendChild(heart);

      visibleHeartCount++;
    }

    /*
      Update the number of hearts based on
      the total number of elapsed days
    */
    function updateHearts(totalDays) {
      while (visibleHeartCount < totalDays) {
        createHeart();
      }
    }

    /*
      Update the timer
    */
    function updateTimer() {
      const now = new Date();

      let difference =
        now.getTime() - startDate.getTime();

      if (difference < 0) {
        yearsElement.textContent = "0";
        daysElement.textContent = "0";
        hoursElement.textContent = "0";
        minutesElement.textContent = "0";
        secondsElement.textContent = "0";
        totalDaysElement.textContent = "0";

        return;
      }

      const totalSeconds =
        Math.floor(difference / 1000);

      const secondsInMinute = 60;
      const secondsInHour = 60 * 60;
      const secondsInDay = 24 * 60 * 60;
      const secondsInYear = 365 * secondsInDay;

      /*
        Total elapsed days.
        This value controls the number of hearts.
      */
      const totalDays =
        Math.floor(totalSeconds / secondsInDay);

      const years =
        Math.floor(totalSeconds / secondsInYear);

      const remainingAfterYears =
        totalSeconds % secondsInYear;

      const days =
        Math.floor(
          remainingAfterYears / secondsInDay
        );

      const remainingAfterDays =
        remainingAfterYears % secondsInDay;

      const hours =
        Math.floor(
          remainingAfterDays / secondsInHour
        );

      const remainingAfterHours =
        remainingAfterDays % secondsInHour;

      const minutes =
        Math.floor(
          remainingAfterHours / secondsInMinute
        );

      const seconds =
        remainingAfterHours % secondsInMinute;

      yearsElement.textContent = years;
      daysElement.textContent = days;
      hoursElement.textContent = hours;
      minutesElement.textContent = minutes;
      secondsElement.textContent = seconds;

      totalDaysElement.textContent =
        totalDays.toLocaleString("en-IE");

      /*
        One elapsed day = one heart
      */
      updateHearts(totalDays);
    }

    updateTimer();

    setInterval(updateTimer, 1000);
  </script>

</body>
</html>