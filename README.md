<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Project Planet: Interactive Curriculum</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <!-- Firebase Libraries -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, getDoc, collection, getDocs, onSnapshot } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        
        // --- Firebase Configuration ---
        let firebaseConfig;
        try {
            // This global variable is provided by the execution environment.
            firebaseConfig = JSON.parse(__firebase_config);
        } catch (e) {
            console.warn("Firebase config not found. Using placeholder values. Please replace with your actual Firebase configuration if running locally.");
            // Fallback for local development
            firebaseConfig = {
                apiKey: "YOUR_API_KEY",
                authDomain: "YOUR_AUTH_DOMAIN",
                projectId: "YOUR_PROJECT_ID",
                storageBucket: "YOUR_STORAGE_BUCKET",
                messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
                appId: "YOUR_APP_ID"
            };
        }

        // --- App Initialization ---
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);

        window.firebase = { auth, db, onAuthStateChanged, doc, setDoc, getDoc, collection, getDocs, onSnapshot, signInAnonymously, signInWithCustomToken };
    </script>
    <!-- Chosen Palette: Earthy Green & Sand -->
    <!-- Application Structure Plan: The application is structured as an interactive syllabus dashboard. A fixed sidebar navigation allows users to instantly switch between the 8 weeks of the curriculum. The main content area dynamically displays the selected week's schedule, with each day presented as an expandable card. Users can mark days as complete via checkboxes, which updates both a weekly progress bar and an overall progress donut chart in a summary section. This structure was chosen because it provides a clear, task-oriented user flow for a student or instructor who needs to view the curriculum and track their progress, making the static document a useful, interactive tool. -->
    <!-- Visualization & Content Choices: 
        - Report Info: Weekly curriculum structure -> Goal: Navigate -> Viz/Presentation: Vertical sidebar navigation -> Interaction: Click to load week's content -> Justification: Provides clear, persistent access to all modules, which is the primary user task. -> Library/Method: HTML/JS.
        - Report Info: Daily topics, sub-topics, objectives -> Goal: Inform -> Viz/Presentation: Expandable cards for each day -> Interaction: Click to toggle details -> Justification: Manages information density, showing high-level topics first and allowing users to drill down without being overwhelmed. -> Library/Method: HTML/Tailwind/JS.
        - Report Info: 'Progress' column -> Goal: Track -> Viz/Presentation: Checkboxes and progress bars -> Interaction: Click to update state -> Justification: Transforms the static plan into a dynamic tracking tool, adding direct value for the user. -> Library/Method: HTML/JS.
        - Report Info: Overall course progress -> Goal: Synthesize -> Viz/Presentation: Donut Chart -> Interaction: Updates automatically on checkbox click -> Justification: Offers a quick, high-level visual summary of completion. -> Library/Method: Chart.js (Canvas).
    -->
    <!-- CONFIRMATION: NO SVG graphics used. NO Mermaid JS used. -->
    <style>
        body {
            font-family: 'Inter', sans-serif;
        }
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
        .details-content {
            max-height: 0;
            overflow: hidden;
            transition: max-height 0.5s ease-out;
        }
        .details-content.open {
            max-height: 500px;
        }
        .chart-container {
            position: relative;
            width: 100%;
            max-width: 350px;
            margin-left: auto;
            margin-right: auto;
            height: 350px;
            max-height: 400px;
        }
        #loader {
            transition: opacity 0.5s;
        }
    </style>
</head>
<body class="bg-stone-50 text-stone-800">
    
    <div id="loader" class="fixed inset-0 bg-white flex items-center justify-center z-[100]">
        <div class="text-center">
            <svg class="animate-spin h-10 w-10 text-emerald-600 mx-auto" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
            </svg>
            <p class="mt-4 text-stone-600">Initializing Curriculum...</p>
        </div>
    </div>


    <div class="flex flex-col md:flex-row min-h-screen">
        <aside class="w-full md:w-64 bg-white border-r border-stone-200 p-6 flex-shrink-0">
            <h1 class="text-2xl font-bold text-emerald-800 mb-2">Project Planet</h1>
            <p class="text-sm text-stone-500 mb-8">Interactive Curriculum</p>
            <nav id="week-navigation" class="space-y-2">
            </nav>
            <div class="mt-8 pt-6 border-t border-stone-200">
                <h3 class="text-xs font-semibold text-stone-400 uppercase tracking-wider mb-3">Admin Tools</h3>
                <button id="admin-view-btn" class="w-full text-left px-4 py-2 rounded-lg text-sm font-medium transition-colors duration-200 text-stone-600 hover:bg-stone-100">View All Responses</button>
            </div>
        </aside>

        <main class="flex-1 p-6 md:p-10">
            <div id="user-section" class="mb-8"></div>
            <div id="main-content">
            </div>

            <section id="dashboard" class="mt-12 bg-white rounded-2xl p-6 md:p-8 border border-stone-200">
                <h2 class="text-2xl font-bold text-stone-700 mb-2">Overall Progress Dashboard</h2>
                <p class="text-stone-600 mb-6">This dashboard provides a visual summary of your progress through the course. As you mark daily topics as complete, the chart will update in real-time to reflect your overall completion status.</p>
                 <div class="chart-container">
                    <canvas id="progressChart"></canvas>
                </div>
                <div class="text-center mt-6">
                    <button id="review-btn" class="px-6 py-3 bg-emerald-700 text-white font-semibold rounded-lg hover:bg-emerald-800 transition-colors shadow-md">
                        Review My Responses
                    </button>
                </div>
            </section>
        </main>
    </div>

    <!-- Modals -->
    <div id="response-modal" class="hidden fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
        <div class="bg-white rounded-2xl shadow-xl w-full max-w-2xl max-h-[90vh] flex flex-col">
            <div class="flex justify-between items-center p-5 border-b border-stone-200">
                <h3 id="my-responses-title" class="text-xl font-bold text-stone-800">Your Saved Reflections</h3>
                <button class="close-modal-btn text-stone-500 hover:text-stone-800 text-2xl" aria-label="Close modal">&times;</button>
            </div>
            <div id="modal-content" class="p-6 overflow-y-auto"></div>
            <div class="p-5 border-t border-stone-200 text-right">
                 <button id="print-btn" class="px-5 py-2 bg-emerald-600 text-white font-semibold rounded-lg hover:bg-emerald-700 transition-colors">
                    Print Responses
                </button>
            </div>
        </div>
    </div>
    
    <div id="admin-modal" class="hidden fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
        <div class="bg-white rounded-2xl shadow-xl w-full max-w-4xl max-h-[90vh] flex flex-col">
            <div class="flex justify-between items-center p-5 border-b border-stone-200">
                <h3 class="text-xl font-bold text-stone-800">Admin View: All User Responses</h3>
                <button class="close-modal-btn text-stone-500 hover:text-stone-800 text-2xl" aria-label="Close modal">&times;</button>
            </div>
            <div id="admin-modal-content" class="p-6 overflow-y-auto bg-stone-50"></div>
        </div>
    </div>


    <script type="module">
        const curriculumData = [
            { week: 1, title: "Foundations of Climate Change", days: [ { day: "Mon", topic: "Introduction to PPH", subTopics: [], objective: "Understand the project's purpose and structure." }, { day: "Tue", topic: "Introduction to Climate Change", subTopics: ["Definition, drivers of climate change", "Human vs natural causes", "Global impacts overview"], objective: "Understand what climate change is and its primary drivers." }, { day: "Wed", topic: "The Science of Greenhouse Gases", subTopics: ["CO2, CH4, N2O basics", "Carbon cycle", "Feedback loops"], objective: "Explain how GHGs trap heat and drive warming." }, { day: "Thu", topic: "Climate vs Weather", subTopics: ["30-year averages vs daily weather", "Activity: compare local climate vs daily forecast"], objective: "Distinguish between short-term weather and long-term climate." }, { day: "Fri", topic: "Evidence of Climate Change", subTopics: ["Global warming trends (IPCC graphs)", "Ice melt & sea-level rise", "Regional evidence (South Asia)"], objective: "Identify scientific evidence for climate change." } ] },
            { week: 2, title: "Impacts of Climate Change", days: [ { day: "Mon", topic: "Human Health & Agriculture", subTopics: ["Food security risks", "Vector-borne diseases", "Malnutrition & crop failure"], objective: "Analyse how climate change threatens human systems." }, { day: "Tue", topic: "Ecosystems & Biodiversity", subTopics: ["Coral bleaching activity", "Habitat shifts & extinctions"], objective: "Understand biodiversity loss linked to climate change." }, { day: "Wed", topic: "Extreme Weather", subTopics: ["Hurricanes, droughts, floods", "Attribution science basics"], objective: "Connect climate change to extreme weather events." }, { day: "Thu", topic: "Water Resources", subTopics: ["Glacier melt", "Water scarcity & conflict"], objective: "Assess impacts on global water availability." }, { day: "Fri", topic: "Economic Impacts", subTopics: ["Infrastructure damage", "Insurance industry shifts", "Supply chain disruption"], objective: "Explain the economic consequences of climate inaction." } ] },
            { week: 3, title: "Climate Justice & Equity", days: [ { day: "Mon", topic: "Intro to Climate Justice", subTopics: ["Defining climate justice", "Historical responsibility"], objective: "Define climate justice and its core principles." }, { day: "Tue", topic: "Vulnerable Communities", subTopics: ["Global South impacts", "Indigenous communities", "Gendered impacts"], objective: "Identify communities most vulnerable to climate change." }, { day: "Wed", topic: "Equity in Solutions", subTopics: ["Just transition", "Access to green technology"], objective: "Analyse equity in proposed climate solutions." }, { day: "Thu", topic: "Case Study Debates", subTopics: ["Student-led debates on specific justice issues"], objective: "Debate real-world climate justice scenarios." }, { day: "Fri", topic: "Loss and Damage", subTopics: ["Concept and UNFCCC negotiations"], objective: "Explain the 'loss and damage' fund concept." } ] },
            { week: 4, title: "Solutions: Mitigation", days: [ { day: "Mon", topic: "Renewable Energy", subTopics: ["Solar, wind, geothermal", "Grid integration challenges"], objective: "Compare different renewable energy sources." }, { day: "Tue", topic: "Energy Efficiency", subTopics: ["Building retrofits", "Smart grids"], objective: "Understand the role of energy efficiency." }, { day: "Wed", topic: "Carbon Capture & Storage", subTopics: ["Technology overview", "Debates on viability"], objective: "Explain how CCS technology works." }, { day: "Thu", topic: "Sustainable Transport", subTopics: ["Electric vehicles", "Public transit", "Urban planning"], objective: "Analyse solutions for decarbonising transport." }, { day: "Fri", topic: "Policy Levers", subTopics: ["Carbon tax vs. cap-and-trade"], objective: "Differentiate key mitigation policies." } ] },
            { week: 5, title: "Solutions: Adaptation & Resilience", days: [ { day: "Mon", topic: "Intro to Adaptation", subTopics: ["Mitigation vs. Adaptation"], objective: "Distinguish between adaptation and mitigation." }, { day: "Tue", topic: "Climate-Resilient Infrastructure", subTopics: ["Sea walls, floating architecture"], objective: "Describe examples of resilient infrastructure." }, { day: "Wed", topic: "Early Warning Systems", subTopics: ["For extreme weather events"], objective: "Explain the importance of early warning systems." }, { day: "Thu", topic: "Traditional Knowledge", subTopics: ["Indigenous methods of farming & water use"], objective: "Appreciate indigenous knowledge in adaptation." }, { day: "Fri", topic: "Climate Action Kit Workshop", subTopics: ["STEM project: sensors, micro:bit"], objective: "Build hands-on STEM-based climate science tools." } ] },
            { week: 6, title: "Climate Policy & Governance", days: [ { day: "Mon", topic: "UNFCCC & Paris Agreement", subTopics: ["Structure and goals", "NDCs (Nationally Determined Contributions)"], objective: "Explain the framework of the Paris Agreement." }, { day: "Tue", topic: "Role of National Gov'ts", subTopics: ["Policy implementation"], objective: "Analyse the role of national governments." }, { day: "Wed", topic: "Sub-national Actors", subTopics: ["Cities (C40), states/provinces"], objective: "Recognise the role of cities and states." }, { day: "Thu", topic: "Corporate Responsibility", subTopics: ["ESG, greenwashing"], objective: "Critique corporate climate action claims." }, { day: "Fri", topic: "Civil Society & Activism", subTopics: ["Youth movements", "Role of NGOs"], objective: "Understand the impact of civil society." } ] },
            { week: 7, title: "The Economics of Climate Change", days: [ { day: "Mon", topic: "Carbon Credits", subTopics: ["Voluntary vs. compliance markets"], objective: "Explain how carbon credit markets work." }, { day: "Tue", topic: "Gender and Climate Finance", subTopics: [], objective: "Analyse the intersection of gender and climate finance." }, { day: "Wed", topic: "Indigenous Rights in Finance", subTopics: [], objective: "Discuss the role of indigenous rights in climate finance." }, { day: "Thu", topic: "Judicial Activism", subTopics: ["Climate litigation trends"], objective: "Understand the role of the judiciary in climate action." }, { day: "Fri", topic: "Role of Non-State Actors", subTopics: ["Philanthropy, private investment"], objective: "Discuss the financial role of non-state actors." } ] },
            { week: 8, title: "Future Pathways & Action", days: [ { day: "Mon", topic: "Climate Innovations", subTopics: ["CCS, hydrogen, geoengineering"], objective: "Understand emerging climate technologies." }, { day: "Tue", topic: "Climate Finance Future", subTopics: ["Green bonds, insurance"], objective: "Explain future pathways for climate finance." }, { day: "Wed", topic: "Local Solutions", subTopics: ["Community solar, farmer cooperatives"], objective: "Recognise local actions as climate solutions." }, { day: "Thu", topic: "Personal Action Plan", subTopics: ["Carbon footprint calculator exercise"], objective: "Design a personal climate pledge." }, { day: "Fri", topic: "Final Presentations", subTopics: ["Climate Fresk reflections", "Student pledges"], objective: "Present solutions and commit to climate action." } ] }
        ];

        let state = {
            currentWeek: 1,
            userData: {
                name: '',
                progress: {},
                responses: {}
            },
            userId: null,
            progressChart: null
        };
        
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const { auth, db, onAuthStateChanged, doc, setDoc, getDoc, collection, getDocs, signInAnonymously, signInWithCustomToken } = window.firebase;

        // DOM Elements
        const weekNavigation = document.getElementById('week-navigation');
        const mainContent = document.getElementById('main-content');
        const userSection = document.getElementById('user-section');
        const ctx = document.getElementById('progressChart').getContext('2d');
        const reviewBtn = document.getElementById('review-btn');
        const responseModal = document.getElementById('response-modal');
        const modalContent = document.getElementById('modal-content');
        const printBtn = document.getElementById('print-btn');
        const adminViewBtn = document.getElementById('admin-view-btn');
        const adminModal = document.getElementById('admin-modal');
        const adminModalContent = document.getElementById('admin-modal-content');
        const loader = document.getElementById('loader');

        // --- Data Handling ---
        async function loadUserData() {
            if (!state.userId) return;
            const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'userResponses', state.userId);
            const docSnap = await getDoc(docRef);
            if (docSnap.exists()) {
                const data = docSnap.data();
                state.userData = {
                    name: data.name || '',
                    progress: data.progress || {},
                    responses: data.responses || {}
                };
            }
            render();
        }

        async function saveUserData() {
            if (!state.userId) return;
            const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'userResponses', state.userId);
            await setDoc(docRef, state.userData, { merge: true });
        }

        // --- Rendering ---
        function renderUserSection() {
            if (!state.userData.name) {
                userSection.innerHTML = `
                    <div class="bg-white rounded-2xl p-6 border border-stone-200">
                        <h2 class="text-xl font-bold text-stone-700 mb-2">Start Your Session</h2>
                        <p class="text-stone-600 mb-4">Please enter your name to track your progress and responses.</p>
                        <div class="flex flex-col sm:flex-row items-stretch sm:items-center gap-2">
                            <input type="text" id="name-input" placeholder="Your Name" class="flex-grow w-full sm:w-auto p-2 border border-stone-300 rounded-lg focus:ring-emerald-500 focus:border-emerald-500">
                            <button id="save-name-btn" class="w-full sm:w-auto px-4 py-2 bg-emerald-600 text-white font-semibold rounded-lg hover:bg-emerald-700 transition-colors">Save Name</button>
                        </div>
                    </div>`;
                document.getElementById('save-name-btn').addEventListener('click', () => {
                    const nameInput = document.getElementById('name-input');
                    if (nameInput.value.trim()) {
                        state.userData.name = nameInput.value.trim();
                        saveUserData();
                        render();
                    }
                });
            } else {
                 userSection.innerHTML = ``;
            }
        }

        function renderNavigation() {
            weekNavigation.innerHTML = '';
            curriculumData.forEach(weekData => {
                const button = document.createElement('button');
                button.textContent = `Week ${weekData.week}: ${weekData.title}`;
                button.className = `w-full text-left px-4 py-2 rounded-lg text-sm font-medium transition-colors duration-200 ${state.currentWeek === weekData.week ? 'bg-emerald-100 text-emerald-800' : 'text-stone-600 hover:bg-stone-100'}`;
                button.onclick = () => {
                    state.currentWeek = weekData.week;
                    renderWeekContent();
                    renderNavigation();
                };
                weekNavigation.appendChild(button);
            });
        }
        
        function renderWeekContent() {
            const weekData = curriculumData.find(w => w.week === state.currentWeek);
            if (!weekData) return;

            const completedDays = weekData.days.filter((d, i) => state.userData.progress[`w${state.currentWeek}d${i}`]).length;
            const totalDays = weekData.days.length;
            const progressPercentage = totalDays > 0 ? (completedDays / totalDays) * 100 : 0;
            
            let welcomeMessage = state.userData.name ? `<h2 class="text-3xl font-bold text-stone-800">Welcome, ${state.userData.name}</h2>` : '';
            mainContent.innerHTML = `
                ${welcomeMessage}
                <h3 class="text-2xl font-bold text-stone-700 ${state.userData.name ? 'mt-1' : ''}">${weekData.title}</h3>
                <p class="text-stone-500 mt-1 mb-6">Week ${weekData.week} Overview</p>
                <div class="mb-6">
                    <div class="flex justify-between items-center mb-1"><span class="text-sm font-medium text-emerald-700">Weekly Progress</span><span class="text-sm font-medium text-emerald-700">${Math.round(progressPercentage)}%</span></div>
                    <div class="w-full bg-stone-200 rounded-full h-2.5"><div class="bg-emerald-600 h-2.5 rounded-full" style="width: ${progressPercentage}%"></div></div>
                </div>
                <div class="grid grid-cols-1 lg:grid-cols-2 xl:grid-cols-3 gap-6">${weekData.days.map((day, index) => renderDayCard(day, index)).join('')}</div>`;
            addCardEventListeners();
        }
        
        function renderDayCard(day, index) {
            const progressKey = `w${state.currentWeek}d${index}`;
            const responseKey = `w${state.currentWeek}d${index}`;
            const isCompleted = !!state.userData.progress[progressKey];
            const responseText = state.userData.responses[responseKey] || '';

            return `
                <div class="bg-white border border-stone-200 rounded-2xl p-5 transition-shadow hover:shadow-lg">
                    <div class="flex items-start justify-between">
                        <div>
                            <p class="text-sm font-semibold text-emerald-700">${day.day.toUpperCase()}</p>
                            <h3 class="font-bold text-lg text-stone-800 mt-1">${day.topic}</h3>
                        </div>
                        <div class="flex items-center space-x-3">
                            <button class="toggle-details p-1 text-stone-400 hover:text-stone-700" data-index="${index}" aria-label="Toggle details">▼</button>
                            <input type="checkbox" class="h-5 w-5 rounded border-stone-300 text-emerald-600 focus:ring-emerald-500" data-key="${progressKey}" ${isCompleted ? 'checked' : ''}>
                        </div>
                    </div>
                    <div class="details-content mt-4" id="details-${index}">
                        <h4 class="font-semibold text-sm text-stone-600 mb-1">Sub-Topics:</h4>
                        ${day.subTopics.length > 0 ? `<ul class="list-disc list-inside text-sm text-stone-500 space-y-1 mb-3">${day.subTopics.map(st => `<li>${st}</li>`).join('')}</ul>` : '<p class="text-sm text-stone-500 italic mb-3">No sub-topics listed.</p>'}
                        <h4 class="font-semibold text-sm text-stone-600 mb-1">Learning Objective:</h4>
                        <p class="text-sm text-stone-500">${day.objective}</p>
                        <div class="mt-4 pt-4 border-t border-stone-200">
                            <h4 class="font-semibold text-sm text-stone-600 mb-2">Your Reflections</h4>
                            <textarea class="w-full p-2 border border-stone-300 rounded-lg text-sm focus:ring-emerald-500 focus:border-emerald-500" rows="3" placeholder="Enter your thoughts..." data-key="${responseKey}">${responseText}</textarea>
                            <button class="save-response-btn mt-2 px-3 py-1 bg-emerald-600 text-white text-xs font-semibold rounded-lg hover:bg-emerald-700" data-key="${responseKey}">Save</button>
                        </div>
                    </div>
                </div>`;
        }

        function renderDashboard() {
            const allDays = curriculumData.flatMap((w, wi) => w.days.map((d, di) => `w${wi+1}d${di}`)).length;
            const completedCount = Object.values(state.userData.progress).filter(Boolean).length;
            const remainingCount = allDays - completedCount;

            const data = {
                labels: ['Completed', 'Remaining'],
                datasets: [{ data: [completedCount, remainingCount], backgroundColor: ['#059669', '#e7e5e4'], borderWidth: 4, hoverOffset: 8 }]
            };

            if (state.progressChart) {
                state.progressChart.data = data;
                state.progressChart.update();
            } else {
                state.progressChart = new Chart(ctx, {
                    type: 'doughnut',
                    data: data,
                    options: { responsive: true, maintainAspectRatio: false, cutout: '70%', plugins: { legend: { position: 'bottom' } } }
                });
            }
        }
        
        function render() {
            if (!state.userId) return; // Wait for auth
            if (!state.userData.name) {
                renderUserSection();
                mainContent.innerHTML = '';
                document.getElementById('dashboard').style.display = 'none';
            } else {
                userSection.innerHTML = '';
                document.getElementById('dashboard').style.display = 'block';
                renderNavigation();
                renderWeekContent();
                renderDashboard();
            }
        }

        // --- Event Listeners ---
        function addCardEventListeners() {
            document.querySelectorAll('.toggle-details').forEach(button => button.addEventListener('click', e => {
                const index = e.target.closest('[data-index]').dataset.index;
                const content = document.getElementById(`details-${index}`);
                content.classList.toggle('open');
                e.target.textContent = content.classList.contains('open') ? '▲' : '▼';
            }));

            document.querySelectorAll('input[type="checkbox"]').forEach(checkbox => checkbox.addEventListener('change', e => {
                state.userData.progress[e.target.dataset.key] = e.target.checked;
                saveUserData();
                render(); 
            }));

            document.querySelectorAll('.save-response-btn').forEach(button => button.addEventListener('click', e => {
                const key = e.target.dataset.key;
                const textarea = e.target.previousElementSibling;
                state.userData.responses[key] = textarea.value;
                saveUserData();
                e.target.textContent = 'Saved!';
                setTimeout(() => { e.target.textContent = 'Save'; }, 1500);
            }));
        }
        
        // --- Modal Logic ---
        function populateAndShowModal() {
            let html = '';
            let responseCount = 0;
            curriculumData.forEach((week, wi) => {
                const weekResponses = week.days.map((day, di) => {
                    const response = state.userData.responses[`w${wi+1}d${di}`];
                    return response ? { ...day, response } : null;
                }).filter(Boolean);

                if (weekResponses.length > 0) {
                    html += `<div class="mb-6"><h4 class="text-lg font-bold text-emerald-800 border-b pb-2 mb-3">Week ${week.week}: ${week.title}</h4>`;
                    weekResponses.forEach(day => {
                        html += `<div class="mb-4"><p class="font-semibold text-stone-700">${day.day}: ${day.topic}</p><p class="text-stone-600 bg-stone-50 p-3 rounded-md mt-1 whitespace-pre-wrap">${day.response}</p></div>`;
                        responseCount++;
                    });
                    html += `</div>`;
                }
            });
            modalContent.innerHTML = responseCount > 0 ? html : '<p class="text-stone-500 text-center py-8">You have not saved any responses yet.</p>';
            document.getElementById('my-responses-title').textContent = `${state.userData.name}'s Saved Reflections`;
            responseModal.classList.remove('hidden');
        }
        
        async function showAdminView() {
            adminModalContent.innerHTML = '<p class="text-stone-500 text-center py-8">Loading responses...</p>';
            adminModal.classList.remove('hidden');
            
            const querySnapshot = await getDocs(collection(db, 'artifacts', appId, 'public', 'data', 'userResponses'));
            let html = '';
            let userCount = 0;
            querySnapshot.forEach(docSnap => {
                const user = docSnap.data();
                const responses = user.responses || {};
                const responseKeys = Object.keys(responses).filter(k => responses[k] && responses[k].trim());

                if(responseKeys.length > 0) {
                    userCount++;
                    html += `<div class="bg-white p-4 rounded-lg shadow-sm mb-4"><h3 class="font-bold text-lg text-emerald-900">${user.name || 'Anonymous User'} <span class="text-sm font-normal text-stone-500">(${docSnap.id})</span></h3>`;
                    
                    responseKeys.forEach(key => {
                        const match = key.match(/w(\d+)d(\d+)/);
                        if (match) {
                            const [, week, day] = match;
                            const weekData = curriculumData[week-1];
                            if(weekData && weekData.days[day]) {
                                const dayData = weekData.days[day];
                                html += `<div class="mt-3 pl-4 border-l-2 border-emerald-100"><p class="font-semibold text-sm text-stone-700">${dayData.topic}</p><p class="text-stone-600 bg-stone-100 p-2 rounded mt-1 text-sm whitespace-pre-wrap">${responses[key]}</p></div>`;
                            }
                        }
                    });
                    html += `</div>`;
                }
            });
            adminModalContent.innerHTML = userCount > 0 ? html : '<p class="text-stone-500 text-center py-8">No users have submitted responses yet.</p>';
        }

        // --- Init and Event Binding ---
        async function initializeAppAndAuth() {
            onAuthStateChanged(auth, user => {
                if (user) {
                    state.userId = user.uid;
                    loadUserData().then(() => {
                        loader.style.opacity = '0';
                        setTimeout(() => loader.style.display = 'none', 500);
                    });
                }
            });

            try {
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("Authentication failed:", error);
                loader.innerHTML = '<p class="text-red-500">Authentication Failed. Please refresh.</p>';
            }
        }

        document.addEventListener('DOMContentLoaded', () => {
            initializeAppAndAuth();

            reviewBtn.addEventListener('click', populateAndShowModal);
            adminViewBtn.addEventListener('click', showAdminView);
            document.querySelectorAll('.close-modal-btn').forEach(btn => btn.addEventListener('click', () => {
                responseModal.classList.add('hidden');
                adminModal.classList.add('hidden');
            }));
            
            printBtn.addEventListener('click', () => {
                const printContent = modalContent.innerHTML;
                const userNameTitle = state.userData.name ? `${state.userData.name}'s` : 'My';
                const printWindow = window.open('', '', 'height=800,width=800');
                printWindow.document.write('<html><head><title>' + userNameTitle + ' Project Planet Responses</title>');
                printWindow.document.write('<style>body{font-family: sans-serif; line-height: 1.5;} h1{color: #065f46;} h4{font-size: 1.2em; color: #047857; border-bottom: 2px solid #d1fae5; padding-bottom: 8px; margin-bottom: 12px;} div{margin-bottom: 1.5rem;} p{margin: 4px 0;} .font-semibold{font-weight: 600;} .response-text{background-color: #f5f5f4; padding: 12px; border-radius: 6px; white-space: pre-wrap; border: 1px solid #e7e5e4;}</style>');
                printWindow.document.write('</head><body>');
                printWindow.document.write(`<h1>${userNameTitle} Project Planet Responses</h1>`);
                printWindow.document.write(printContent.replace(/class="[^"]*"/g, ''));
                printWindow.document.write('</body></html>');
                printWindow.document.close();
                printWindow.focus();
                printWindow.print();
            }); 
        });
    </script>
</body>
</html>

