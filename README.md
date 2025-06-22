<!DOCTYPE html>
<html lang="ar">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>تحدي القراءة متعدد اللغات</title>
  <style>
    body {
      background-color: #0d1117;
      color: white;
      font-family: 'Tahoma', sans-serif;
      text-align: center;
      padding: 20px;
      margin: 0;
      position: relative;
    }

    h1 {
      font-size: 24px;
      margin-bottom: 20px;
    }

    #language {
      position: absolute;
      top: 20px;
      right: 20px;
      padding: 8px 12px;
      font-size: 16px;
      background-color: #00aaff;
      color: white;
      border: none;
      border-radius: 8px;
      cursor: pointer;
    }

    #language:hover {
      background-color: #0088cc;
    }

    #counter {
      margin: 10px 0;
      font-size: 18px;
      color: #00ffaa;
    }

    .word {
      font-size: 22px;
      margin: 4px;
      display: inline-block;
      padding: 6px 8px;
      border-radius: 8px;
      transition: background-color 0.3s, color 0.3s, transform 0.3s;
    }

    .read {
      background-color: #00aaff;
      color: #fff;
      animation: pop 0.3s ease;
    }

    .current {
      border: 2px dashed #00ffaa;
      background: linear-gradient(45deg, #00ffaa33, #00aaff33);
      animation: blink 1s infinite;
    }

    @keyframes pop {
      0% { transform: scale(1); }
      50% { transform: scale(1.2); }
      100% { transform: scale(1); }
    }

    @keyframes blink {
      0%, 100% { opacity: 1; }
      50% { opacity: 0.3; }
    }

    button {
      font-size: 18px;
      padding: 10px 16px;
      margin: 10px 5px;
      border: none;
      border-radius: 10px;
      background-color: #00aaff;
      color: white;
      cursor: pointer;
      width: 90%;
      max-width: 300px;
    }

    button:hover {
      background-color: #0088cc;
    }

    #text {
      margin: 20px auto;
      max-width: 90%;
      line-height: 2;
    }

    #status {
      margin-top: 15px;
      font-size: 18px;
      color: #ffaa00;
    }

    audio {
      display: none;
    }
  </style>
</head>
<body>

  <select id="language" onchange="changeLanguage()">
    <option value="ar">🇸🇦 العربية</option>
    <option value="en">🇺🇸 English</option>
    <option value="es">🇪🇸 Español</option>
    <option value="fr">🇫🇷 Français</option>
  </select>

  <h1>📢 اقرأ الكلمة التالية بصوتك</h1>

  <div id="counter">قرأت 0 كلمة</div>
  <div id="text"></div>

  <button onclick="startListening()">🎙️ ابدأ القراءة</button>
  <button onclick="resetReading()">🔁 إعادة</button>
  <button onclick="changeSentence()">🔀 جملة جديدة</button>

  <div id="status"></div>

  <audio id="ding" src="https://cdn.pixabay.com/download/audio/2021/08/04/audio_b4b9dc4904.mp3?filename=correct-2-46134.mp3"></audio>
  <audio id="wrong" src="https://cdn.pixabay.com/download/audio/2022/03/15/audio_1f4bc94c33.mp3?filename=error-126627.mp3"></audio>
  <audio id="done" src="https://cdn.pixabay.com/download/audio/2021/10/26/audio_d1a455b0d6.mp3?filename=correct-answer-4-204118.mp3"></audio>

  <script>
    const allSentences = {
      ar: [
        "كل نجاح يبدأ بخطوة فلا تتردد",
        "الصبر مفتاح الفرج والثقة طريق الراحة",
        "المستحيل مجرد رأي وليس حقيقة",
        "الفشل هو بداية الطريق إلى النجاح",
        "التعليم هو السلاح الأقوى الذي يمكنك استخدامه لتغيير العالم",
        "العقل زينة والحكمة مفتاح النجاح",
        "القراءة توسّع المدارك وتبني العقول"
      ],
      en: [
        "Success is a journey not a destination",
        "Hard work beats talent when talent does not work hard",
        "The future depends on what you do today",
        "Believe you can and you're halfway there",
        "Reading expands the mind and opens doors to success"
      ],
      es: [
        "El éxito comienza con un paso",
        "Nunca renuncies a tus sueños",
        "El fracaso es el primer paso hacia el éxito",
        "Cree en ti mismo y sigue adelante",
        "La lectura te hace libre y sabio"
      ],
      fr: [
        "Le succès commence par un pas",
        "N'abandonne jamais tes rêves",
        "L'échec est la première étape vers le succès",
        "Crois en toi et avance",
        "La lecture ouvre l'esprit et nourrit l'âme"
      ]
    };

    const langs = {
      ar: "ar-SA",
      en: "en-US",
      es: "es-ES",
      fr: "fr-FR"
    };

    let currentText = "";
    let currentLang = "ar";
    let currentWordIndex = 0;
    let spans = [];

    const textContainer = document.getElementById("text");
    const counter = document.getElementById("counter");
    const sound = document.getElementById("ding");
    const wrongSound = document.getElementById("wrong");
    const doneSound = document.getElementById("done");
    const languageSelect = document.getElementById("language");
    const status = document.getElementById("status");

    let recognition;
    let isListening = false;

    function normalize(text) {
      return text
        .normalize("NFD").replace(/[\u064B-\u065F\u0610-\u061A\u06D6-\u06ED]/g, "")
        .replace(/[إأآا]/g, "ا")
        .replace(/ء/g, "")
        .replace(/[يى]/g, "ي")
        .replace(/[^a-zA-Zأ-ي0-9 ]/g, "")
        .trim().toLowerCase();
    }

    function compareWords(w1, w2) {
      const a = normalize(w1);
      const b = normalize(w2);
      if (a === b) return 1;
      let matches = 0;
      const minLen = Math.min(a.length, b.length);
      for (let i = 0; i < minLen; i++) {
        if (a[i] === b[i]) matches++;
      }
      return matches / Math.max(a.length, b.length);
    }

    function displayText(text) {
      textContainer.innerHTML = "";
      currentWordIndex = 0;
      const words = text.split(" ");
      spans = [];

      words.forEach((word, index) => {
        const span = document.createElement("span");
        span.className = "word";
        span.textContent = word;
        if (index === 0) span.classList.add("current");
        spans.push(span);
        textContainer.appendChild(span);
      });

      updateCounter();
    }

    function updateCounter() {
      const readCount = document.querySelectorAll(".read").length;
      const total = spans.length;
      const messages = {
        ar: `قرأت ${readCount} من ${total} كلمة`,
        en: `You read ${readCount} of ${total} words`,
        es: `Leíste ${readCount} de ${total} palabras`,
        fr: `Vous avez lu ${readCount} sur ${total} mots`
      };
      counter.textContent = messages[currentLang];
    }

    function setupRecognition() {
      recognition = new webkitSpeechRecognition();
      recognition.lang = langs[currentLang];
      recognition.continuous = true;
      recognition.interimResults = true;

      recognition.onresult = function(event) {
        const transcript = event.results[event.results.length - 1][0].transcript.trim();
        const spoken = normalize(transcript);
        const currentWord = normalize(spans[currentWordIndex]?.textContent || "");
        const similarity = compareWords(spoken, currentWord);

        if (similarity >= 0.8) {
          spans[currentWordIndex].classList.remove("current");
          spans[currentWordIndex].classList.add("read");
          sound.play();
          currentWordIndex++;

          if (spans[currentWordIndex]) {
            spans[currentWordIndex].classList.add("current");
          } else {
            status.textContent = currentLang === "ar" ? "🎉 أحسنت! انتهيت من الجملة" :
              currentLang === "en" ? "🎉 Well done! Sentence completed" :
              currentLang === "es" ? "🎉 ¡Bien hecho! Frase completada" :
              "🎉 Bravo ! Phrase terminée";
            doneSound.play();
            recognition.stop();
            isListening = false;
          }

          updateCounter();
        } else {
          wrongSound.play();
          status.textContent = currentLang === "ar"
            ? `❌ لم يتم التعرف على الكلمة بدقة: "${transcript}"`
            : `❌ Not accurate: "${transcript}"`;
        }
      };

      recognition.onerror = () => restartRecognition();
      recognition.onend = () => { if (isListening) restartRecognition(); };
    }

    function startListening() {
      if (isListening) return;
      isListening = true;
      status.textContent = currentLang === "ar" ? "🎧 جاري الاستماع..." :
        currentLang === "en" ? "🎧 Listening..." :
        currentLang === "es" ? "🎧 Escuchando..." :
        "🎧 Écoute en cours...";
      setupRecognition();
      recognition.start();
    }

    function restartRecognition() {
      if (recognition) recognition.stop();
      setupRecognition();
      recognition.start();
    }

    function resetReading() {
      spans.forEach(span => span.classList.remove("read", "current"));
      if (spans[0]) spans[0].classList.add("current");
      currentWordIndex = 0;
      updateCounter();
      status.textContent = "";
      if (recognition) recognition.stop();
      isListening = false;
    }

    function changeSentence() {
      const list = allSentences[currentLang];
      const randomIndex = Math.floor(Math.random() * list.length);
      currentText = list[randomIndex];
      resetReading();
      displayText(currentText);
    }

    function changeLanguage() {
      currentLang = languageSelect.value;
      document.body.dir = currentLang === "ar" ? "rtl" : "ltr";
      changeSentence();
    }

    window.onload = () => {
      currentText = allSentences[currentLang][0];
      displayText(currentText);
      document.body.dir = currentLang === "ar" ? "rtl" : "ltr";
    };
  </script>
</body>
</html><!DOCTYPE html>
<html lang="ar">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>تحدي القراءة متعدد اللغات</title>
  <style>
    body {
      background-color: #0d1117;
      color: white;
      font-family: 'Tahoma', sans-serif;
      text-align: center;
      padding: 20px;
      margin: 0;
      position: relative;
    }

    h1 {
      font-size: 24px;
      margin-bottom: 20px;
    }

    #language {
      position: absolute;
      top: 20px;
      right: 20px;
      padding: 8px 12px;
      font-size: 16px;
      background-color: #00aaff;
      color: white;
      border: none;
      border-radius: 8px;
      cursor: pointer;
    }

    #language:hover {
      background-color: #0088cc;
    }

    #counter {
      margin: 10px 0;
      font-size: 18px;
      color: #00ffaa;
    }

    .word {
      font-size: 22px;
      margin: 4px;
      display: inline-block;
      padding: 6px 8px;
      border-radius: 8px;
      transition: background-color 0.3s, color 0.3s, transform 0.3s;
    }

    .read {
      background-color: #00aaff;
      color: #fff;
      animation: pop 0.3s ease;
    }

    .current {
      border: 2px dashed #00ffaa;
      background: linear-gradient(45deg, #00ffaa33, #00aaff33);
      animation: blink 1s infinite;
    }

    @keyframes pop {
      0% { transform: scale(1); }
      50% { transform: scale(1.2); }
      100% { transform: scale(1); }
    }

    @keyframes blink {
      0%, 100% { opacity: 1; }
      50% { opacity: 0.3; }
    }

    button {
      font-size: 18px;
      padding: 10px 16px;
      margin: 10px 5px;
      border: none;
      border-radius: 10px;
      background-color: #00aaff;
      color: white;
      cursor: pointer;
      width: 90%;
      max-width: 300px;
    }

    button:hover {
      background-color: #0088cc;
    }

    #text {
      margin: 20px auto;
      max-width: 90%;
      line-height: 2;
    }

    #status {
      margin-top: 15px;
      font-size: 18px;
      color: #ffaa00;
    }

    audio {
      display: none;
    }
  </style>
</head>
<body>

  <select id="language" onchange="changeLanguage()">
    <option value="ar">🇸🇦 العربية</option>
    <option value="en">🇺🇸 English</option>
    <option value="es">🇪🇸 Español</option>
    <option value="fr">🇫🇷 Français</option>
  </select>

  <h1>📢 اقرأ الكلمة التالية بصوتك</h1>

  <div id="counter">قرأت 0 كلمة</div>
  <div id="text"></div>

  <button onclick="startListening()">🎙️ ابدأ القراءة</button>
  <button onclick="resetReading()">🔁 إعادة</button>
  <button onclick="changeSentence()">🔀 جملة جديدة</button>

  <div id="status"></div>

  <audio id="ding" src="https://cdn.pixabay.com/download/audio/2021/08/04/audio_b4b9dc4904.mp3?filename=correct-2-46134.mp3"></audio>
  <audio id="wrong" src="https://cdn.pixabay.com/download/audio/2022/03/15/audio_1f4bc94c33.mp3?filename=error-126627.mp3"></audio>
  <audio id="done" src="https://cdn.pixabay.com/download/audio/2021/10/26/audio_d1a455b0d6.mp3?filename=correct-answer-4-204118.mp3"></audio>

  <script>
    const allSentences = {
      ar: [
        "كل نجاح يبدأ بخطوة فلا تتردد",
        "الصبر مفتاح الفرج والثقة طريق الراحة",
        "المستحيل مجرد رأي وليس حقيقة",
        "الفشل هو بداية الطريق إلى النجاح",
        "التعليم هو السلاح الأقوى الذي يمكنك استخدامه لتغيير العالم",
        "العقل زينة والحكمة مفتاح النجاح",
        "القراءة توسّع المدارك وتبني العقول"
      ],
      en: [
        "Success is a journey not a destination",
        "Hard work beats talent when talent does not work hard",
        "The future depends on what you do today",
        "Believe you can and you're halfway there",
        "Reading expands the mind and opens doors to success"
      ],
      es: [
        "El éxito comienza con un paso",
        "Nunca renuncies a tus sueños",
        "El fracaso es el primer paso hacia el éxito",
        "Cree en ti mismo y sigue adelante",
        "La lectura te hace libre y sabio"
      ],
      fr: [
        "Le succès commence par un pas",
        "N'abandonne jamais tes rêves",
        "L'échec est la première étape vers le succès",
        "Crois en toi et avance",
        "La lecture ouvre l'esprit et nourrit l'âme"
      ]
    };

    const langs = {
      ar: "ar-SA",
      en: "en-US",
      es: "es-ES",
      fr: "fr-FR"
    };

    let currentText = "";
    let currentLang = "ar";
    let currentWordIndex = 0;
    let spans = [];

    const textContainer = document.getElementById("text");
    const counter = document.getElementById("counter");
    const sound = document.getElementById("ding");
    const wrongSound = document.getElementById("wrong");
    const doneSound = document.getElementById("done");
    const languageSelect = document.getElementById("language");
    const status = document.getElementById("status");

    let recognition;
    let isListening = false;

    function normalize(text) {
      return text
        .normalize("NFD").replace(/[\u064B-\u065F\u0610-\u061A\u06D6-\u06ED]/g, "")
        .replace(/[إأآا]/g, "ا")
        .replace(/ء/g, "")
        .replace(/[يى]/g, "ي")
        .replace(/[^a-zA-Zأ-ي0-9 ]/g, "")
        .trim().toLowerCase();
    }

    function compareWords(w1, w2) {
      const a = normalize(w1);
      const b = normalize(w2);
      if (a === b) return 1;
      let matches = 0;
      const minLen = Math.min(a.length, b.length);
      for (let i = 0; i < minLen; i++) {
        if (a[i] === b[i]) matches++;
      }
      return matches / Math.max(a.length, b.length);
    }

    function displayText(text) {
      textContainer.innerHTML = "";
      currentWordIndex = 0;
      const words = text.split(" ");
      spans = [];

      words.forEach((word, index) => {
        const span = document.createElement("span");
        span.className = "word";
        span.textContent = word;
        if (index === 0) span.classList.add("current");
        spans.push(span);
        textContainer.appendChild(span);
      });

      updateCounter();
    }

    function updateCounter() {
      const readCount = document.querySelectorAll(".read").length;
      const total = spans.length;
      const messages = {
        ar: `قرأت ${readCount} من ${total} كلمة`,
        en: `You read ${readCount} of ${total} words`,
        es: `Leíste ${readCount} de ${total} palabras`,
        fr: `Vous avez lu ${readCount} sur ${total} mots`
      };
      counter.textContent = messages[currentLang];
    }

    function setupRecognition() {
      recognition = new webkitSpeechRecognition();
      recognition.lang = langs[currentLang];
      recognition.continuous = true;
      recognition.interimResults = true;

      recognition.onresult = function(event) {
        const transcript = event.results[event.results.length - 1][0].transcript.trim();
        const spoken = normalize(transcript);
        const currentWord = normalize(spans[currentWordIndex]?.textContent || "");
        const similarity = compareWords(spoken, currentWord);

        if (similarity >= 0.8) {
          spans[currentWordIndex].classList.remove("current");
          spans[currentWordIndex].classList.add("read");
          sound.play();
          currentWordIndex++;

          if (spans[currentWordIndex]) {
            spans[currentWordIndex].classList.add("current");
          } else {
            status.textContent = currentLang === "ar" ? "🎉 أحسنت! انتهيت من الجملة" :
              currentLang === "en" ? "🎉 Well done! Sentence completed" :
              currentLang === "es" ? "🎉 ¡Bien hecho! Frase completada" :
              "🎉 Bravo ! Phrase terminée";
            doneSound.play();
            recognition.stop();
            isListening = false;
          }

          updateCounter();
        } else {
          wrongSound.play();
          status.textContent = currentLang === "ar"
            ? `❌ لم يتم التعرف على الكلمة بدقة: "${transcript}"`
            : `❌ Not accurate: "${transcript}"`;
        }
      };

      recognition.onerror = () => restartRecognition();
      recognition.onend = () => { if (isListening) restartRecognition(); };
    }

    function startListening() {
      if (isListening) return;
      isListening = true;
      status.textContent = currentLang === "ar" ? "🎧 جاري الاستماع..." :
        currentLang === "en" ? "🎧 Listening..." :
        currentLang === "es" ? "🎧 Escuchando..." :
        "🎧 Écoute en cours...";
      setupRecognition();
      recognition.start();
    }

    function restartRecognition() {
      if (recognition) recognition.stop();
      setupRecognition();
      recognition.start();
    }

    function resetReading() {
      spans.forEach(span => span.classList.remove("read", "current"));
      if (spans[0]) spans[0].classList.add("current");
      currentWordIndex = 0;
      updateCounter();
      status.textContent = "";
      if (recognition) recognition.stop();
      isListening = false;
    }

    function changeSentence() {
      const list = allSentences[currentLang];
      const randomIndex = Math.floor(Math.random() * list.length);
      currentText = list[randomIndex];
      resetReading();
      displayText(currentText);
    }

    function changeLanguage() {
      currentLang = languageSelect.value;
      document.body.dir = currentLang === "ar" ? "rtl" : "ltr";
      changeSentence();
    }

    window.onload = () => {
      currentText = allSentences[currentLang][0];
      displayText(currentText);
      document.body.dir = currentLang === "ar" ? "rtl" : "ltr";
    };
  </script>
</body>
</html>
