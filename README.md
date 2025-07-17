<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>نظام متابعة المهام</title>
    <meta name="theme-color" content="#3b82f6"/>
    <link rel="manifest" href="manifest.json">
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Tajawal:wght@400;500;700&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Tajawal', sans-serif; background-color: #f4f7f6; }
        .view { display: none; }
        .view.active { display: block; }
        #app-container, #login-container { display: none; }
        .modal-backdrop { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background-color: rgba(0,0,0,0.6); z-index: 40; }
        .modal { display: none; position: fixed; top: 50%; left: 50%; transform: translate(-50%, -50%); z-index: 50; max-height: 90vh; overflow-y: auto; }
        .loader { border: 4px solid #f3f3f3; border-top: 4px solid #3498db; border-radius: 50%; width: 40px; height: 40px; animation: spin 1s linear infinite; margin: 20px auto; }
        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
    </style>
</head>
<body class="p-4 md:p-6">

    <div id="loading-screen" class="text-center p-10">
        <div class="loader"></div>
        <p>جاري تهيئة التطبيق...</p>
    </div>

    <div id="login-container" class="max-w-md mx-auto mt-20">
        <div id="loginView" class="bg-white p-8 rounded-xl shadow-lg">
            <h2 class="text-2xl font-bold text-center text-gray-800 mb-6">تسجيل الدخول</h2>
            <form id="loginForm">
                <div class="mb-4">
                    <label for="username" class="block text-gray-700 text-sm font-bold mb-2">اسم المستخدم:</label>
                    <select id="username" class="shadow-sm bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5"></select>
                </div>
                <div class="mb-6">
                    <label for="password" class="block text-gray-700 text-sm font-bold mb-2">كلمة المرور:</label>
                    <input type="password" id="password" class="shadow-sm bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5" required>
                </div>
                <button type="submit" class="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg transition-colors">دخول</button>
                <p id="loginError" class="text-red-500 text-xs text-center mt-4"></p>
            </form>
            <div id="passwordHint" class="mt-4 p-3 bg-gray-100 rounded-lg text-xs text-center"></div>
        </div>
    </div>

    <div id="app-container" class="max-w-7xl mx-auto">
        <div class="mb-6 p-4 bg-white rounded-xl shadow-md flex flex-col md:flex-row items-center justify-between">
            <h1 class="text-2xl md:text-3xl font-bold text-gray-800 mb-4 md:mb-0">نظام متابعة المهام</h1>
            <div class="flex items-center">
                <span id="welcomeMessage" class="ml-4 font-semibold text-gray-700"></span>
                <button id="logoutBtn" class="bg-red-500 hover:bg-red-600 text-white font-bold py-2 px-4 rounded-lg text-sm">تسجيل الخروج</button>
            </div>
        </div>
        <div class="mb-4 border-b border-gray-200"><ul id="navTabs" class="flex flex-wrap -mb-px text-sm font-medium text-center" role="tablist"></ul></div>
        <div id="viewsContainer">
            <div id="managerView" class="view"></div>
            <div id="employeeManagementView" class="view p-6 bg-white rounded-xl shadow-md"></div>
            <div id="employeeView" class="view"></div>
            <div id="dashboardView" class="view"></div>
        </div>
    </div>

    <!-- Modals -->
    <div id="modalBackdrop" class="modal-backdrop"></div>
    <div id="managerModal" class="modal w-11/12 md:w-1/2 lg:w-2/5 bg-white rounded-lg shadow-xl p-6"></div>
    <div id="employeeModal" class="modal w-11/12 md:w-1/2 lg:w-2/5 bg-white rounded-lg shadow-xl p-6"></div>
    <div id="commentModal" class="modal w-11/12 md:w-1/3 bg-white rounded-lg shadow-xl p-6"></div>
    <div id="replyModal" class="modal w-11/12 md:w-1/3 bg-white rounded-lg shadow-xl p-6"></div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, onAuthStateChanged, signInAnonymously } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, doc, addDoc, getDoc, setDoc, onSnapshot, updateDoc, deleteDoc, writeBatch, arrayUnion } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        const firebaseConfig = {
            apiKey: "AIzaSyDL1bogdKWQP6NS1mgSE9LLXBrsR3fx1Y0",
            authDomain: "infath-tasks.firebaseapp.com",
            projectId: "infath-tasks",
            storageBucket: "infath-tasks.appspot.com",
            messagingSenderId: "371939646288",
            appId: "1:371939646288:web:905620f64ec58461a0ea55"
        };
        
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        
        const usersCollectionRef = collection(db, 'users');
        const tasksCollectionRef = collection(db, 'tasks');

        let loggedInUser = null;
        let tasksUnsubscribe = null;
        let usersUnsubscribe = null;
        let localUsersCache = {};
        
        const dom = {
            loadingScreen: document.getElementById('loading-screen'),
            loginContainer: document.getElementById('login-container'),
            appContainer: document.getElementById('app-container'),
            loginForm: document.getElementById('loginForm'),
            loginError: document.getElementById('loginError'),
            welcomeMessage: document.getElementById('welcomeMessage'),
            logoutBtn: document.getElementById('logoutBtn'),
            navTabs: document.getElementById('navTabs'),
            views: {
                manager: document.getElementById('managerView'),
                employee: document.getElementById('employeeView'),
                dashboard: document.getElementById('dashboardView'),
                employeeManagement: document.getElementById('employeeManagementView'),
            },
            modals: {
                backdrop: document.getElementById('modalBackdrop'),
                manager: document.getElementById('managerModal'),
                employee: document.getElementById('employeeModal'),
                comment: document.getElementById('commentModal'),
                reply: document.getElementById('replyModal'),
            }
        };

        // --- Core Functions ---

        async function initializeDefaultUsers() {
            const defaultUsers = {
                manager: { password: 'manager123', role: 'manager', name: 'المدير' },
                ahmed: { password: 'ahmed456', role: 'employee', name: 'أحمد' },
                noura: { password: 'noura789', role: 'employee', name: 'نورة' }
            };
            try {
                const managerDocRef = doc(usersCollectionRef, 'manager');
                const docSnap = await getDoc(managerDocRef);
                if (!docSnap.exists()) {
                    const batch = writeBatch(db);
                    Object.entries(defaultUsers).forEach(([username, data]) => {
                        const userDocRef = doc(usersCollectionRef, username);
                        batch.set(userDocRef, data);
                    });
                    await batch.commit();
                }
            } catch (e) {
                throw e; 
            }
        }

        function listenToUsers() {
            if (usersUnsubscribe) usersUnsubscribe();
            usersUnsubscribe = onSnapshot(usersCollectionRef, (snapshot) => {
                localUsersCache = {};
                snapshot.forEach(doc => { localUsersCache[doc.id] = doc.data(); });
                populateLoginUsers();
                if (loggedInUser?.role === 'manager') renderEmployeeList();
            }, (error) => console.error("Error listening to users:", error));
        }

        function fetchAndRenderTasks() {
            if (tasksUnsubscribe) tasksUnsubscribe();
            tasksUnsubscribe = onSnapshot(tasksCollectionRef, (querySnapshot) => {
                const allTasks = [];
                querySnapshot.forEach((doc) => allTasks.push({ id: doc.id, ...doc.data() }));
                
                renderDashboardView(allTasks);
                if (loggedInUser?.role === 'manager') renderManagerView(allTasks);
                if (loggedInUser?.role === 'employee') {
                    const myTasks = allTasks.filter(t => t.assignedTo === loggedInUser.username);
                    renderEmployeeView(myTasks);
                }
            }, (error) => console.error("Error fetching tasks: ", error));
        }

        // --- UI Rendering ---

        function populateLoginUsers() {
            const usernameSelect = document.getElementById('username');
            const passwordHint = document.getElementById('passwordHint');
            if (!usernameSelect || !passwordHint) return;
            const selectedUser = usernameSelect.value;
            usernameSelect.innerHTML = '';
            passwordHint.innerHTML = '<p><strong>كلمات المرور للتجربة:</strong></p>';
            Object.entries(localUsersCache).forEach(([username, data]) => {
                const option = document.createElement('option');
                option.value = username;
                option.textContent = data.name;
                usernameSelect.appendChild(option);
                passwordHint.innerHTML += `<p>${data.name} (${username}): ${data.password}</p>`;
            });
            if (selectedUser) usernameSelect.value = selectedUser;
        }

        function renderDashboardView(tasks) {
            const container = dom.views.dashboard;
            container.innerHTML = `<h2 class="text-xl font-bold text-gray-700 mb-5">لوحة التقدم العامة</h2><div id="dashboardTasksContainer" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6"></div>`;
            tasks.forEach(task => document.getElementById('dashboardTasksContainer').appendChild(renderTaskCard(task, 'dashboard')));
        }
        
        function renderManagerView(tasks) {
            const container = dom.views.manager;
            container.innerHTML = `<div class="flex justify-between items-center mb-5">
                                       <h2 class="text-xl font-bold text-gray-700">مهام الفريق</h2>
                                       <button id="managerAddTaskBtn" class="bg-blue-600 text-white font-bold py-2 px-5 rounded-lg shadow-md hover:bg-blue-700 transition-colors">+ إسناد مهمة جديدة</button>
                                   </div>
                                   <div id="managerTasksContainer" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6"></div>`;
            document.getElementById('managerAddTaskBtn').addEventListener('click', () => openModal('manager'));
            tasks.forEach(task => document.getElementById('managerTasksContainer').appendChild(renderTaskCard(task, 'manager')));
        }

        function renderEmployeeView(tasks) {
            const container = dom.views.employee;
            container.innerHTML = `<h2 class="text-xl font-bold text-gray-700 mb-5">المهام المسندة إليك</h2><div id="employeeTasksContainer" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6"></div>`;
            tasks.forEach(task => document.getElementById('employeeTasksContainer').appendChild(renderTaskCard(task, 'employee')));
        }
        
        function renderTaskCard(task, viewType) {
            const card = document.createElement('div');
            const completedSubtasks = task.subTasks?.filter(st => st.completed).length || 0;
            const totalSubtasks = task.subTasks?.length || 0;
            const progress = totalSubtasks > 0 ? (completedSubtasks / totalSubtasks) * 100 : 0;

            let statusText, statusColor;
            if (progress === 100) { statusText = 'مكتملة'; statusColor = 'green'; }
            else if (new Date(task.managerDueDate) < new Date() && progress < 100) { statusText = 'متأخرة'; statusColor = 'red'; }
            else if (progress > 0 || totalSubtasks > 0) { statusText = 'قيد التنفيذ'; statusColor = 'yellow'; }
            else { statusText = 'لم تبدأ'; statusColor = 'gray'; }
            
            card.className = `task-card bg-white p-5 rounded-xl shadow-lg border-t-8 border-${statusColor}-500 flex flex-col`;
            
            const subtasksHtml = (task.subTasks && totalSubtasks > 0) ? task.subTasks.map((st, index) => {
                let commentHtml = '';
                if (st.managerComment) {
                    commentHtml = `<div class="mt-2 p-2 bg-yellow-50 rounded-md text-xs">
                        <p><strong class="text-yellow-800">ملاحظة المدير:</strong> ${st.managerComment}</p>
                        ${st.employeeReply ? `<div class="mt-1 pt-1 border-t border-yellow-200"><strong class="text-green-800">رد الموظف:</strong> ${st.employeeReply}</div>` : ''}
                    </div>`;
                }

                let actionButton = '';
                if (viewType === 'manager' && !st.managerComment) {
                    actionButton = `<button data-task-id="${task.id}" data-subtask-index="${index}" class="add-comment-btn text-xs text-blue-600 hover:underline">إضافة ملاحظة</button>`;
                } else if (viewType === 'employee' && st.managerComment && !st.employeeReply) {
                    actionButton = `<button data-task-id="${task.id}" data-subtask-index="${index}" data-comment="${st.managerComment}" class="add-reply-btn text-xs text-green-600 hover:underline">الرد</button>`;
                }

                return `<div class="py-2 border-b last:border-b-0">
                    <label class="flex items-start">
                        <input type="checkbox" class="form-checkbox h-5 w-5 text-blue-600 rounded mt-1" 
                               data-task-id="${task.id}" data-subtask-index="${index}" 
                               ${st.completed ? 'checked' : ''} ${viewType !== 'employee' || task.assignedTo !== loggedInUser.username ? 'disabled' : ''}>
                        <div class="flex-grow mr-3">
                           <span class="ml-2 ${st.completed ? 'line-through text-gray-400' : 'text-gray-800'}">${st.text}</span>
                           <div class="flex justify-between items-center">
                             <span class="text-xs text-gray-400">${st.employeeDueDate}</span>
                             ${actionButton}
                           </div>
                           ${commentHtml}
                        </div>
                    </label>
                </div>`;
            }).join('') : '<div class="text-sm text-gray-500 mt-3">لم يتم إضافة خطوات.</div>';
            
            let cardButtonsHtml = '';
            if (viewType === 'manager') {
                cardButtonsHtml = `<button class="delete-task-btn text-red-500 hover:text-red-700 text-sm font-semibold" data-task-id="${task.id}">حذف</button>`;
            } else if (viewType === 'employee' && task.assignedTo === loggedInUser.username) {
                 cardButtonsHtml = `<button class="add-subtask-btn bg-green-100 text-green-800 hover:bg-green-200 text-xs font-bold py-2 px-3 rounded-lg" 
                            data-task-id="${task.id}" data-due-date="${task.managerDueDate}">+ إضافة خطوة</button>`;
            }
            
            card.innerHTML = `<div class="flex-grow">
                    <div class="flex justify-between items-start mb-2">
                        <h3 class="font-bold text-lg text-gray-900">${task.taskName}</h3>
                        <span class="text-xs font-bold px-3 py-1 rounded-full bg-${statusColor}-100 text-${statusColor}-800">${statusText}</span>
                    </div>
                    <p class="text-sm text-gray-600">${task.description || ''}</p>
                    <p class="text-xs text-gray-500 mt-2">المسؤول: ${localUsersCache[task.assignedTo]?.name || task.assignedTo} | التسليم: ${task.managerDueDate}</p>
                    <div class="mt-4 border-t pt-3">
                        <h4 class="text-sm font-bold text-gray-600 mb-2">الخطوات التنفيذية:</h4>
                        <div class="space-y-1">${subtasksHtml}</div>
                    </div>
                </div>
                <div class="mt-4 pt-4 border-t flex justify-between items-center">
                     <div class="w-full">
                        <div class="text-xs text-gray-500 mb-1">التقدم: ${Math.round(progress)}%</div>
                        <div class="w-full bg-gray-200 rounded-full h-2.5">
                            <div class="bg-${statusColor}-500 h-2.5 rounded-full transition-all duration-500" style="width: ${progress}%"></div>
                        </div>
                    </div>
                    <div class="mr-4 flex-shrink-0">${cardButtonsHtml}</div>
                </div>`;

            card.querySelector('.delete-task-btn')?.addEventListener('click', (e) => deleteTask(e.target.dataset.taskId));
            card.querySelector('.add-subtask-btn')?.addEventListener('click', (e) => openModal('employee', e.target.dataset));
            card.querySelectorAll('.add-comment-btn').forEach(btn => btn.addEventListener('click', e => openModal('comment', e.target.dataset)));
            card.querySelectorAll('.add-reply-btn').forEach(btn => btn.addEventListener('click', e => openModal('reply', e.target.dataset)));
            card.querySelectorAll('input[type="checkbox"]').forEach(cb => cb.addEventListener('change', e => updateSubtaskStatus(e.target.dataset.taskId, parseInt(e.target.dataset.subtaskIndex), e.target.checked)));
            
            return card;
        }

        function renderEmployeeList() {
            const container = dom.views.employeeManagement;
            container.innerHTML = `<h2 class="text-xl font-bold text-gray-700 mb-5">إدارة الموظفين</h2>
                <form id="addEmployeeForm" class="flex items-end gap-4 mb-6 pb-6 border-b">
                    <div class="flex-grow"><label for="newEmployeeUsername" class="block text-gray-700 text-sm font-bold mb-2">اسم المستخدم (انجليزي):</label><input type="text" id="newEmployeeUsername" placeholder="e.g., khalid" class="shadow-sm bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg w-full p-2.5" required></div>
                    <div class="flex-grow"><label for="newEmployeeName" class="block text-gray-700 text-sm font-bold mb-2">الاسم المعروض:</label><input type="text" id="newEmployeeName" placeholder="e.g., خالد" class="shadow-sm bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg w-full p-2.5" required></div>
                    <div class="flex-grow"><label for="newEmployeePassword" class="block text-gray-700 text-sm font-bold mb-2">كلمة المرور:</label><input type="text" id="newEmployeePassword" placeholder="كلمة مرور قوية" class="shadow-sm bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg w-full p-2.5" required></div>
                    <button type="submit" class="bg-green-600 text-white font-bold py-2 px-4 rounded-lg h-10">+ إضافة</button>
                </form>
                <div id="employeeList" class="space-y-2"></div>`;
            
            document.getElementById('addEmployeeForm').addEventListener('submit', handleAddEmployee);

            const listEl = document.getElementById('employeeList');
            listEl.innerHTML = '';
            Object.entries(localUsersCache).filter(([_, data]) => data.role === 'employee').forEach(([username, data]) => {
                const div = document.createElement('div');
                div.className = 'flex justify-between items-center p-3 bg-gray-50 rounded-lg';
                div.innerHTML = `<div><p class="font-semibold">${data.name}</p><p class="text-sm text-gray-500">@${username}</p></div>
                                 <button data-username="${username}" class="delete-employee-btn text-red-500 hover:text-red-700 text-sm">حذف</button>`;
                listEl.appendChild(div);
            });
            
            listEl.querySelectorAll('.delete-employee-btn').forEach(btn => btn.addEventListener('click', (e) => deleteEmployee(e.target.dataset.username)));
        }
        
        function openModal(type, data = {}) {
            closeAllModals();
            let modal;
            if (type === 'manager') {
                modal = dom.modals.manager;
                modal.innerHTML = ` <h2 class="text-xl font-bold mb-4">إسناد مهمة جديدة</h2>
                                    <form id="managerForm">
                                        <div class="mb-4"><label for="taskName" class="block text-gray-700 text-sm font-bold mb-2">اسم المهمة:</label><input type="text" id="taskName" class="shadow-sm bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5" required></div>
                                        <div class="mb-4"><label for="taskDescription" class="block text-gray-700 text-sm font-bold mb-2">شرح المهمة:</label><textarea id="taskDescription" rows="3" class="shadow-sm bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5"></textarea></div>
                                        <div class="grid grid-cols-2 gap-4 mb-4">
                                            <div><label for="assignedTo" class="block text-gray-700 text-sm font-bold mb-2">إسناد إلى:</label><select id="assignedTo" class="shadow-sm bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5" required></select></div>
                                            <div><label for="managerDueDate" class="block text-gray-700 text-sm font-bold mb-2">تاريخ التسليم النهائي:</label><input type="date" id="managerDueDate" class="shadow-sm bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5" required></div>
                                        </div>
                                        <div class="flex items-center justify-end mt-6"><button type="button" class="modal-cancel-btn bg-gray-300 hover:bg-gray-400 text-gray-800 font-bold py-2 px-4 rounded-lg ml-2">إلغاء</button><button type="submit" class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg">حفظ المهمة</button></div>
                                    </form>`;
                const assignToSelect = modal.querySelector('#assignedTo');
                Object.entries(localUsersCache).filter(([_, u]) => u.role === 'employee').forEach(([username, u]) => {
                    const option = document.createElement('option');
                    option.value = username;
                    option.textContent = u.name;
                    assignToSelect.appendChild(option);
                });
                modal.querySelector('#managerForm').addEventListener('submit', handleManagerTaskForm);
            } else if (type === 'employee') {
                modal = dom.modals.employee;
                modal.innerHTML = `<h2 class="text-xl font-bold mb-4">إضافة خطوة تنفيذية</h2>
                                   <form id="employeeForm"><input type="hidden" id="subtaskParentId" value="${data.taskId}">
                                       <div class="mb-4"><label for="subtaskName" class="block text-gray-700 text-sm font-bold mb-2">وصف الخطوة:</label><input type="text" id="subtaskName" class="shadow-sm bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5" required></div>
                                       <div class="mb-4"><label for="employeeDueDate" class="block text-gray-700 text-sm font-bold mb-2">تاريخ انتهاء الخطوة:</label><input type="date" id="employeeDueDate" max="${data.dueDate}" class="shadow-sm bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5" required><p class="text-xs text-gray-500 mt-1">يجب أن لا يتجاوز تاريخ تسليم المهمة الرئيسي.</p></div>
                                       <div class="flex items-center justify-end mt-6"><button type="button" class="modal-cancel-btn bg-gray-300 hover:bg-gray-400 text-gray-800 font-bold py-2 px-4 rounded-lg ml-2">إلغاء</button><button type="submit" class="bg-green-600 hover:bg-green-700 text-white font-bold py-2 px-4 rounded-lg">إضافة الخطوة</button></div>
                                   </form>`;
                modal.querySelector('#employeeForm').addEventListener('submit', handleEmployeeSubtaskForm);
            } else if (type === 'comment') {
                modal = dom.modals.comment;
                modal.innerHTML = `<h2 class="text-xl font-bold mb-4">إضافة ملاحظة</h2>
                                   <form id="commentForm"><input type="hidden" id="commentTaskId" value="${data.taskId}"><input type="hidden" id="commentSubtaskIndex" value="${data.subtaskIndex}">
                                       <textarea id="commentText" rows="4" class="w-full p-2 border rounded-lg" required placeholder="اكتب ملاحظتك هنا..."></textarea>
                                       <div class="flex justify-end mt-4"><button type="button" class="modal-cancel-btn bg-gray-300 text-gray-800 font-bold py-2 px-4 rounded-lg ml-2">إلغاء</button><button type="submit" class="bg-blue-600 text-white font-bold py-2 px-4 rounded-lg">حفظ</button></div>
                                   </form>`;
                modal.querySelector('#commentForm').addEventListener('submit', handleCommentForm);
            } else if (type === 'reply') {
                modal = dom.modals.reply;
                modal.innerHTML = `<h2 class="text-xl font-bold mb-4">الرد على الملاحظة</h2>
                                   <form id="replyForm"><input type="hidden" id="replyTaskId" value="${data.taskId}"><input type="hidden" id="replySubtaskIndex" value="${data.subtaskIndex}">
                                       <div class="mb-2 p-3 bg-yellow-50 rounded-lg"><p class="text-sm font-bold">ملاحظة المدير:</p><p id="managerCommentText" class="text-sm text-gray-700">${data.comment}</p></div>
                                       <textarea id="replyText" rows="4" class="w-full p-2 border rounded-lg" placeholder="اكتب ردك هنا..." required></textarea>
                                       <div class="flex justify-end mt-4"><button type="button" class="modal-cancel-btn bg-gray-300 text-gray-800 font-bold py-2 px-4 rounded-lg ml-2">إلغاء</button><button type="submit" class="bg-green-600 text-white font-bold py-2 px-4 rounded-lg">إرسال الرد</button></div>
                                   </form>`;
                modal.querySelector('#replyForm').addEventListener('submit', handleReplyForm);
            }
            modal.style.display = 'block';
            modal.querySelectorAll('.modal-cancel-btn').forEach(btn => btn.addEventListener('click', closeAllModals));
            dom.modals.backdrop.style.display = 'block';
        }

        function closeAllModals() {
            Object.values(dom.modals).forEach(m => m.style.display = 'none');
        }

        async function handleAddEmployee(e) {
            e.preventDefault();
            const username = e.target.elements.newEmployeeUsername.value.trim();
            const name = e.target.elements.newEmployeeName.value.trim();
            const password = e.target.elements.newEmployeePassword.value.trim();
            if (!username || !name || !password || !/^[a-zA-Z0-9]+$/.test(username)) {
                alert('يرجى ملء جميع الحقول. اسم المستخدم يجب أن يكون باللغة الإنجليزية بدون مسافات.');
                return;
            }
            if (localUsersCache[username]) {
                alert('اسم المستخدم موجود بالفعل.');
                return;
            }
            await setDoc(doc(usersCollectionRef, username), { name, password, role: 'employee' });
            alert('تمت إضافة الموظف بنجاح.');
            e.target.reset();
        }
        
        async function handleManagerTaskForm(e) {
            e.preventDefault();
            await addDoc(tasksCollectionRef, {
                taskName: e.target.elements.taskName.value,
                description: e.target.elements.taskDescription.value,
                assignedTo: e.target.elements.assignedTo.value,
                managerDueDate: e.target.elements.managerDueDate.value,
                subTasks: []
            });
            closeAllModals();
        }
        
        async function handleEmployeeSubtaskForm(e) {
            e.preventDefault();
            const taskDocRef = doc(tasksCollectionRef, e.target.elements.subtaskParentId.value);
            await updateDoc(taskDocRef, {
                subTasks: arrayUnion({
                    text: e.target.elements.subtaskName.value,
                    employeeDueDate: e.target.elements.employeeDueDate.value,
                    completed: false,
                    managerComment: null,
                    employeeReply: null,
                })
            });
            closeAllModals();
        }
        
        async function handleCommentForm(e) {
            e.preventDefault();
            await updateSubtaskProperty(e.target.elements.commentTaskId.value, parseInt(e.target.elements.commentSubtaskIndex.value), 'managerComment', e.target.elements.commentText.value);
            closeAllModals();
        }
        
        async function handleReplyForm(e) {
            e.preventDefault();
            await updateSubtaskProperty(e.target.elements.replyTaskId.value, parseInt(e.target.elements.replySubtaskIndex.value), 'employeeReply', e.target.elements.replyText.value);
            closeAllModals();
        }

        async function updateSubtaskProperty(taskId, subtaskIndex, property, value) {
            const taskDocRef = doc(tasksCollectionRef, taskId);
            const docSnap = await getDoc(taskDocRef);
            if (docSnap.exists()) {
                const updatedSubtasks = [...docSnap.data().subTasks];
                if(updatedSubtasks[subtaskIndex] != null) {
                    updatedSubtasks[subtaskIndex][property] = value;
                    await updateDoc(taskDocRef, { subTasks: updatedSubtasks });
                }
            }
        }
        
        async function updateSubtaskStatus(taskId, subtaskIndex, isCompleted) {
             await updateSubtaskProperty(taskId, subtaskIndex, 'completed', isCompleted);
        }
        
        async function deleteEmployee(username) {
            if (confirm(`هل أنت متأكد من حذف الموظف ${username}؟`)) {
                await deleteDoc(doc(usersCollectionRef, username));
            }
        }

        async function deleteTask(taskId) {
            if (confirm('هل أنت متأكد من حذف هذه المهمة وكل خطواتها؟')) {
                await deleteDoc(doc(tasksCollectionRef, taskId));
            }
        }

        // --- App Startup ---

        function setupGlobalEventListeners() {
            if (dom.loginForm) dom.loginForm.addEventListener('submit', handleLogin);
            dom.logoutBtn.addEventListener('click', handleLogout);
        }
        
        function handleLogin(e) {
            e.preventDefault();
            const username = document.getElementById('username').value;
            const password = document.getElementById('password').value;
            if (localUsersCache[username] && localUsersCache[username].password === password) {
                loggedInUser = { username, ...localUsersCache[username] };
                dom.loginContainer.style.display = 'none';
                dom.appContainer.style.display = 'block';
                setupAppForUser();
            } else {
                dom.loginError.textContent = 'اسم المستخدم أو كلمة المرور غير صحيحة.';
            }
        }

        function handleLogout() {
            loggedInUser = null;
            if (tasksUnsubscribe) tasksUnsubscribe();
            if (usersUnsubscribe) usersUnsubscribe();
            dom.appContainer.style.display = 'none';
            dom.loginContainer.style.display = 'block';
            if(dom.loginForm) dom.loginForm.reset();
            if(dom.loginError) dom.loginError.textContent = '';
        }
        
        function setupAppForUser() {
            dom.welcomeMessage.textContent = `أهلاً بك، ${loggedInUser.name}`;
            setupTabs();
        }
        
        function setupTabs() {
            dom.navTabs.innerHTML = '';
            let tabs = [];
            if (loggedInUser.role === 'manager') {
                tabs = [
                    { id: 'managerView', name: 'مهام الفريق' },
                    { id: 'dashboardView', name: 'لوحة عامة' },
                    { id: 'employeeManagementView', name: 'إدارة الموظفين' }
                ];
            } else {
                 tabs = [
                    { id: 'employeeView', name: 'مهامي' },
                    { id: 'dashboardView', name: 'لوحة عامة' },
                ];
            }

            tabs.forEach((tab, index) => {
                const li = document.createElement('li');
                li.className = 'mr-2';
                li.innerHTML = `<a href="#" data-view="${tab.id}" class="inline-block p-4 border-b-2 rounded-t-lg ${index === 0 ? 'border-blue-600 text-blue-600' : 'border-transparent hover:text-gray-600 hover:border-gray-300'}">${tab.name}</a>`;
                dom.navTabs.appendChild(li);
            });
            
            dom.navTabs.querySelectorAll('a').forEach(tabLink => {
                tabLink.addEventListener('click', (e) => {
                    e.preventDefault();
                    switchView(e.target.dataset.view);
                    dom.navTabs.querySelectorAll('a').forEach(l => l.classList.remove('border-blue-600', 'text-blue-600'));
                    e.target.classList.add('border-blue-600', 'text-blue-600');
                });
            });
            
            switchView(tabs[0].id);
        }

        function switchView(viewId) {
             Object.values(dom.views).forEach(v => v.classList.remove('active'));
             dom.views[viewId.replace('View', '')].classList.add('active');
             fetchAndRenderTasks();
        }
        
        document.addEventListener('DOMContentLoaded', () => {
            setupGlobalEventListeners();
            onAuthStateChanged(auth, async (user) => {
                if (user) {
                    console.log("Firebase Auth state changed, user detected:", user.uid);
                    try {
                        await initializeDefaultUsers();
                        listenToUsers();
                        dom.loadingScreen.style.display = 'none';
                        dom.loginContainer.style.display = 'block';
                    } catch (error) {
                        console.error("Error during app initialization:", error);
                        let errorMessage = `<div class="text-center"><p class="text-red-500 font-bold">فشل تهيئة التطبيق</p>`;
                        if (error.message.includes("offline") || error.code === 'permission-denied') {
                             errorMessage += `<p class="text-sm text-gray-600 mt-2"><b>السبب:</b> قواعد الأمان في قاعدة بياناتك تمنع الوصول.</p><p class="text-sm text-gray-600 mt-1"><b>الحل:</b> اذهب إلى <b>Firestore Database > Rules</b> وتأكد من أن القاعدة هي:</p><pre class="bg-gray-100 p-2 rounded text-left text-xs mt-2"><code>rules_version = '2';\nservice cloud.firestore {\n  match /databases/{database}/documents {\n    match /{document=**} {\n      allow read, write: if request.auth != null;\n    }\n  }\n}</code></pre>`;
                        } else {
                             errorMessage += `<p class="text-sm text-gray-600 mt-2">حدث خطأ غير متوقع.</p>`;
                        }
                        errorMessage += `</div>`;
                        dom.loginView.innerHTML = errorMessage;
                        dom.loadingScreen.style.display = 'none';
                        dom.loginContainer.style.display = 'block';
                    }
                } else {
                    signInAnonymously(auth).catch(error => {
                        console.error("Anonymous sign-in failed:", error);
                        let errorMessage = `<div class="text-center"><p class="text-red-500 font-bold">فشل المصادقة مع الخادم</p>`;
                        if (error.code === 'auth/admin-restricted-operation') {
                            errorMessage += `<p class="text-sm text-gray-600 mt-2"><b>السبب:</b> ميزة "تسجيل الدخول المجهول" (Anonymous) غير مفعلة.</p><p class="text-sm text-gray-600 mt-1"><b>الحل:</b> اذهب إلى <b>Firebase Console > Authentication > Sign-in method</b> وقم بتفعيل <b>Anonymous</b>.</p>`;
                        } else {
                            errorMessage += `<p class="text-sm text-gray-600 mt-2">لا يمكن الاتصال بخوادم المصادقة. تحقق من اتصالك بالإنترنت.</p>`;
                        }
                        errorMessage += `</div>`;
                        dom.loginView.innerHTML = errorMessage;
                        dom.loadingScreen.style.display = 'none';
                        dom.loginContainer.style.display = 'block';
                    });
                }
            });
        });
    </script>
</body>
</html>

