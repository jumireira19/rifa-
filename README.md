<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Gerador de Rifa</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    fontFamily: {
                        inter: ['Inter', 'sans-serif'],
                    },
                }
            }
        }
    </script>
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f0f2f5;
        }
        .container {
            max-width: 90%;
            margin: 0 auto;
            padding: 1.5rem;
        }
        @media (min-width: 768px) {
            .container {
                max-width: 800px; /* Aumentar um pouco para acomodar mais bilhetes */
            }
        }
        /* Estilo para o bilhete vencedor */
        .winning-ticket {
            background-color: #ef4444; /* Cor de fundo vermelha (red-500 do Tailwind) */
            color: #ffffff; /* Texto branco */
            font-weight: bold;
            border: 2px solid #dc2626; /* Borda vermelha mais escura (red-600 do Tailwind) */
            transform: scale(1.1); /* Um pequeno zoom */
            transition: all 0.2s ease-in-out;
        }
        /* Estilo para bilhetes selecionáveis */
        .selectable-ticket {
            display: inline-flex;
            align-items: center;
            justify-content: center;
            background-color: #e0f2fe; /* blue-50 */
            color: #1e40af; /* blue-800 */
            text-xs font-semibold mr-2 mb-2 px-2.5 py-0.5 rounded-full;
            cursor: pointer;
            transition: background-color 0.2s ease-in-out, transform 0.1s ease-in-out;
            min-width: 40px; /* Garante tamanho mínimo */
            height: 24px;
        }
        .selectable-ticket:hover {
            background-color: #bfdbfe; /* blue-100 */
        }
        .selectable-ticket.selected {
            background-color: #2563eb; /* blue-600 */
            color: #ffffff;
            font-weight: bold;
        }
        .loading-overlay {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(255, 255, 255, 0.8);
            display: flex;
            justify-content: center;
            align-items: center;
            z-index: 1000;
        }
        .spinner {
            border: 4px solid rgba(0, 0, 0, 0.1);
            border-left-color: #2563eb;
            border-radius: 50%;
            width: 40px;
            height: 40px;
            animation: spin 1s linear infinite;
        }
        @keyframes spin {
            to { transform: rotate(360deg); }
        }
    </style>
    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, setDoc, deleteDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Variáveis globais para Firebase
        window.firebaseApp = null;
        window.db = null;
        window.auth = null;
        window.userId = null;
        window.isAuthReady = false; // Flag para indicar que a autenticação está pronta

        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-raffle-app';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        // Inicializa Firebase
        window.firebaseApp = initializeApp(firebaseConfig);
        window.db = getFirestore(window.firebaseApp);
        window.auth = getAuth(window.firebaseApp);

        // Listener para o estado de autenticação
        onAuthStateChanged(window.auth, async (user) => {
            if (user) {
                window.userId = user.uid;
                console.log("Usuário autenticado:", window.userId);
            } else {
                // Tenta autenticar anonimamente se não houver token inicial
                if (!initialAuthToken) {
                    try {
                        const anonUser = await signInAnonymously(window.auth);
                        window.userId = anonUser.user.uid;
                        console.log("Autenticado anonimamente:", window.userId);
                    } catch (error) {
                        console.error("Erro ao autenticar anonimamente:", error);
                    }
                } else {
                    // Se houver token inicial, tenta usá-lo
                    try {
                        const customUser = await signInWithCustomToken(window.auth, initialAuthToken);
                        window.userId = customUser.user.uid;
                        console.log("Autenticado com token personalizado:", window.userId);
                    } catch (error) {
                        console.error("Erro ao autenticar com token personalizado:", error);
                    }
                }
            }
            window.isAuthReady = true;
            // Dispara um evento personalizado para indicar que a autenticação está pronta
            document.dispatchEvent(new CustomEvent('firebaseAuthReady'));
        });
    </script>
</head>
<body class="bg-gray-100 flex items-center justify-center min-h-screen">
    <div id="loadingOverlay" class="loading-overlay hidden">
        <div class="spinner"></div>
    </div>

    <div class="container bg-white p-6 rounded-lg shadow-xl border border-gray-200">
        <h1 class="text-3xl font-bold text-center text-gray-800 mb-6">Gerador de Rifa</h1>

        <!-- Exibição do ID do Usuário -->
        <div class="text-center text-gray-600 text-sm mb-4">
            ID do Usuário: <span id="displayUserId" class="font-mono text-gray-800">Carregando...</span>
        </div>

        <!-- Input para o número máximo de bilhetes a serem gerados -->
        <div class="mb-4">
            <label for="maxTicketNumber" class="block text-gray-700 text-sm font-medium mb-2">
                Número máximo de bilhetes disponíveis para seleção (ex: 100):
            </label>
            <input type="number" id="maxTicketNumber" min="1" value="50"
                   class="shadow-sm appearance-none border rounded-lg w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent transition duration-200 ease-in-out">
            <button id="renderSelectableTicketsBtn"
                    class="mt-2 w-full bg-indigo-500 hover:bg-indigo-600 text-white font-bold py-2 px-4 rounded-lg shadow-md transition duration-200 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-opacity-75">
                Gerar Opções de Bilhetes
            </button>
        </div>

        <!-- Container para bilhetes selecionáveis -->
        <div class="mb-6">
            <h2 class="text-xl font-semibold text-gray-800 mb-3">Selecione os Bilhetes para a Rifa:</h2>
            <div id="selectableTicketsContainer" class="bg-gray-50 p-4 rounded-lg border border-gray-200 min-h-[100px] max-h-60 overflow-y-auto flex flex-wrap gap-2">
                <p class="text-gray-500">Gere as opções de bilhetes acima para começar a selecionar.</p>
            </div>
        </div>

        <!-- Botões de Ação -->
        <div class="flex flex-col sm:flex-row justify-between space-y-3 sm:space-y-0 sm:space-x-3 mb-6">
            <button id="confirmSelectionBtn"
                    class="flex-1 bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg shadow-md transition duration-200 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-opacity-75">
                Confirmar Seleção de Bilhetes
            </button>
            <button id="pickWinnerBtn"
                    class="flex-1 bg-green-600 hover:bg-green-700 text-white font-bold py-2 px-4 rounded-lg shadow-md transition duration-200 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-green-500 focus:ring-opacity-75"
                    disabled>
                Sortear Vencedor
            </button>
        </div>

        <!-- Área de Mensagens -->
        <div id="messageBox" class="bg-yellow-100 border border-yellow-400 text-yellow-700 px-4 py-3 rounded-lg relative hidden mb-4" role="alert">
            <span class="block sm:inline"></span>
        </div>

        <!-- Seção para Atribuir Nomes -->
        <div id="nameAssignmentSection" class="mb-6 hidden">
            <h2 class="text-xl font-semibold text-gray-800 mb-3">Atribuir Nomes aos Compradores:</h2>
            <div id="nameAssignmentInputsContainer" class="bg-gray-50 p-4 rounded-lg border border-gray-200 max-h-60 overflow-y-auto">
                <p class="text-gray-500">Selecione e confirme os bilhetes para atribuir nomes.</p>
            </div>
            <button id="saveNamesBtn"
                    class="mt-4 w-full bg-purple-600 hover:bg-purple-700 text-white font-bold py-2 px-4 rounded-lg shadow-md transition duration-200 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-purple-500 focus:ring-opacity-75">
                Salvar Nomes
            </button>
        </div>

        <!-- Seção para Chave Pix -->
        <div id="pixKeySection" class="mb-6">
            <h2 class="text-xl font-semibold text-gray-800 mb-3">Chave Pix para Compra:</h2>
            <div class="mb-4">
                <label for="pixKeyInput" class="block text-gray-700 text-sm font-medium mb-2">
                    Insira a Chave Pix:
                </label>
                <textarea id="pixKeyInput" rows="2" placeholder="Ex: seuemail@example.com ou 123.456.789-00"
                          class="shadow-sm appearance-none border rounded-lg w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent transition duration-200 ease-in-out"></textarea>
            </div>
            <button id="savePixKeyBtn"
                    class="w-full bg-yellow-600 hover:bg-yellow-700 text-white font-bold py-2 px-4 rounded-lg shadow-md transition duration-200 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-yellow-500 focus:ring-opacity-75">
                Salvar Chave Pix
            </button>
            <div id="displayPixKey" class="mt-4 bg-gray-100 p-3 rounded-lg border border-gray-200 text-gray-800 break-words">
                <p class="text-gray-500">Nenhuma chave Pix salva ainda.</p>
            </div>
        </div>

        <!-- Seção de Compartilhamento -->
        <div id="shareSection" class="mb-6">
            <h2 class="text-xl font-semibold text-gray-800 mb-3">Compartilhar Rifa:</h2>
            <button id="generateShareLinkBtn"
                    class="w-full bg-teal-600 hover:bg-teal-700 text-white font-bold py-2 px-4 rounded-lg shadow-md transition duration-200 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-teal-500 focus:ring-opacity-75">
                Gerar Link de Compartilhamento
            </button>
            <div id="shareLinkDisplaySection" class="mt-4 hidden">
                <label for="shareLinkDisplay" class="block text-gray-700 text-sm font-medium mb-2">
                    Link da Rifa Compartilhada:
                </label>
                <input type="text" id="shareLinkDisplay" readonly
                       class="shadow-sm appearance-none border rounded-lg w-full py-2 px-3 text-gray-700 leading-tight bg-gray-100 cursor-text">
                <button id="copyShareLinkBtn"
                        class="mt-2 w-full bg-gray-700 hover:bg-gray-800 text-white font-bold py-2 px-4 rounded-lg shadow-md transition duration-200 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-gray-500 focus:ring-opacity-75">
                    Copiar Link
                </button>
            </div>
        </div>

        <!-- Exibição dos Bilhetes Confirmados para a Rifa -->
        <div class="mb-6">
            <h2 class="text-xl font-semibold text-gray-800 mb-3">Bilhetes na Rifa:</h2>
            <div id="ticketsDisplay" class="bg-gray-50 p-4 rounded-lg border border-gray-200 min-h-[50px] max-h-40 overflow-y-auto flex flex-wrap gap-2">
                <p class="text-gray-500">Nenhum bilhete confirmado para a rifa ainda.</p>
            </div>
        </div>

        <!-- Exibição do Vencedor -->
        <div class="text-center">
            <h2 class="text-2xl font-bold text-gray-800 mb-3">Vencedor:</h2>
            <div id="winnerDisplay" class="bg-purple-100 text-purple-800 text-4xl font-extrabold p-6 rounded-lg shadow-inner border border-purple-300">
                <span class="text-gray-400 text-lg">Ainda não sorteado</span>
            </div>
        </div>

        <!-- Botão de Reset -->
        <div class="mt-6 text-center">
            <button id="resetBtn"
                    class="bg-red-500 hover:bg-red-600 text-white font-bold py-2 px-4 rounded-lg shadow-md transition duration-200 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-red-500 focus:ring-opacity-75">
                Reiniciar
            </button>
        </div>
    </div>

    <script type="module">
        // Import Firebase functions loaded in the global scope
        import { getFirestore, doc, getDoc, setDoc, deleteDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global variables for application state
        let selectableTickets = []; // All clickable tickets (numbers only)
        let selectedTicketsForRaffle = []; // Tickets selected by the user by clicking (numbers only)
        // generatedTickets now stores objects { number: "001", buyer: "Name" }
        let generatedTickets = [];
        let currentWinner = null; // Variable to store the winning ticket (object { number, buyer })
        let pixKey = ''; // New variable to store the Pix key
        let maxTicketNumberSaved = 50; // New variable to store the saved max ticket number
        let isViewingShared = false; // Flag to indicate if currently viewing a shared link
        let currentShareId = null; // Stores the share ID if viewing a shared link

        // References to DOM elements
        const maxTicketNumberInput = document.getElementById('maxTicketNumber');
        const renderSelectableTicketsBtn = document.getElementById('renderSelectableTicketsBtn');
        const selectableTicketsContainer = document.getElementById('selectableTicketsContainer');
        const confirmSelectionBtn = document.getElementById('confirmSelectionBtn');
        const pickWinnerBtn = document.getElementById('pickWinnerBtn');
        const ticketsDisplay = document.getElementById('ticketsDisplay');
        const winnerDisplay = document.getElementById('winnerDisplay');
        const messageBox = document.getElementById('messageBox');
        const resetBtn = document.getElementById('resetBtn');
        const displayUserId = document.getElementById('displayUserId');
        const loadingOverlay = document.getElementById('loadingOverlay');
        const nameAssignmentSection = document.getElementById('nameAssignmentSection');
        const nameAssignmentInputsContainer = document.getElementById('nameAssignmentInputsContainer');
        const saveNamesBtn = document.getElementById('saveNamesBtn');
        const pixKeyInput = document.getElementById('pixKeyInput');
        const savePixKeyBtn = document.getElementById('savePixKeyBtn');
        const displayPixKey = document.getElementById('displayPixKey');
        const generateShareLinkBtn = document.getElementById('generateShareLinkBtn');
        const shareLinkDisplaySection = document.getElementById('shareLinkDisplaySection');
        const shareLinkDisplay = document.getElementById('shareLinkDisplay');
        const copyShareLinkBtn = document.getElementById('copyShareLinkBtn');

        // Firestore paths
        const getPrivateUserDocRef = () => {
            const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-raffle-app';
            const userId = window.userId; // Use the global userId defined in authentication
            if (!userId) {
                console.error("UserID not available for private document reference.");
                return null;
            }
            return doc(window.db, `artifacts/${appId}/users/${userId}/raffleData`, 'userData');
        };

        const getPublicShareDocRef = (shareId) => {
            const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-raffle-app';
            return doc(window.db, `artifacts/${appId}/public/raffleShares`, shareId);
        };

        /**
         * Shows or hides the loading overlay.
         * @param {boolean} show - True to show, False to hide.
         */
        function toggleLoading(show) {
            if (show) {
                loadingOverlay.classList.remove('hidden');
            } else {
                loadingOverlay.classList.add('hidden');
            }
        }

        /**
         * Displays a message in the message box.
         * @param {string} message - The message to display.
         * @param {string} type - The message type (e.g., 'success', 'error', 'warning').
         */
        function showMessage(message, type = 'info') {
            messageBox.querySelector('span').textContent = message;
            messageBox.classList.remove('hidden', 'bg-red-100', 'border-red-400', 'text-red-700', 'bg-green-100', 'border-green-400', 'text-green-700', 'bg-yellow-100', 'border-yellow-400', 'text-yellow-700');
            messageBox.classList.add('block');

            switch (type) {
                case 'error':
                    messageBox.classList.add('bg-red-100', 'border-red-400', 'text-red-700');
                    break;
                case 'success':
                    messageBox.classList.add('bg-green-100', 'border-green-400', 'text-green-700');
                    break;
                case 'warning':
                case 'info':
                default:
                    messageBox.classList.add('bg-yellow-100', 'border-yellow-400', 'text-yellow-700');
                    break;
            }
            // Hide the message after 5 seconds
            setTimeout(() => {
                messageBox.classList.add('hidden');
            }, 5000);
        }

        /**
         * Saves all raffle data (tickets, winner, Pix key, max ticket number) to Firestore.
         * Saves to the private document if not in shared view.
         * @param {boolean} isPublicShare - If true, saves to a public share document.
         * @param {string} shareId - The ID of the public share document.
         */
        async function saveDataToFirestore(isPublicShare = false, shareId = null) {
            if (isViewingShared && !isPublicShare) { // Prevent saving to private from a shared view
                console.warn("Attempted to save private data while viewing a shared link. Operation aborted.");
                return;
            }
            toggleLoading(true);
            try {
                let docRef;
                if (isPublicShare && shareId) {
                    docRef = getPublicShareDocRef(shareId);
                } else {
                    docRef = getPrivateUserDocRef();
                }

                if (docRef) {
                    const dataToSave = {
                        tickets: generatedTickets,
                        winner: currentWinner,
                        pixKey: pixKey,
                        maxTicketNumber: maxTicketNumberSaved
                    };
                    if (isPublicShare) {
                        dataToSave.ownerId = window.userId; // Store owner ID for public shares
                        dataToSave.lastUpdated = new Date();
                    }
                    await setDoc(docRef, dataToSave);
                    console.log(`Dados da rifa salvos no Firestore (${isPublicShare ? 'público' : 'privado'}).`);
                }
            } catch (e) {
                console.error("Erro ao salvar dados da rifa: ", e);
                showMessage("Erro ao salvar dados. Por favor, tente novamente.", "error");
            } finally {
                toggleLoading(false);
            }
        }

        /**
         * Loads all raffle data from Firestore and updates the UI.
         */
        async function loadDataFromFirestore() {
            toggleLoading(true);
            try {
                const urlParams = new URLSearchParams(window.location.search);
                currentShareId = urlParams.get('shareId');

                let docRef;
                if (currentShareId) {
                    isViewingShared = true;
                    docRef = getPublicShareDocRef(currentShareId);
                    console.log("Carregando dados da rifa compartilhada...");
                } else {
                    isViewingShared = false;
                    docRef = getPrivateUserDocRef();
                    console.log("Carregando dados da rifa privada...");
                }

                if (docRef) {
                    const docSnap = await getDoc(docRef);
                    if (docSnap.exists()) {
                        const data = docSnap.data();
                        generatedTickets = data.tickets || [];
                        currentWinner = data.winner || null;
                        pixKey = data.pixKey || '';
                        maxTicketNumberSaved = data.maxTicketNumber || 50;

                        console.log("Dados da rifa carregados do Firestore:", data);
                        showMessage("Dados da rifa carregados com sucesso!", "success");

                        // Update UI with loaded data
                        displayConfirmedTickets(generatedTickets);
                        if (currentWinner) {
                            winnerDisplay.textContent = `${currentWinner.number} (${currentWinner.buyer || 'Sem nome'})`;
                        } else {
                            winnerDisplay.innerHTML = '<span class="text-gray-400 text-lg">Ainda não sorteado</span>';
                        }

                        pixKeyInput.value = pixKey;
                        displayPixKey.textContent = pixKey || 'Nenhuma chave Pix salva ainda.';
                        maxTicketNumberInput.value = maxTicketNumberSaved;

                        // Render selectable tickets and mark selected ones
                        renderSelectableTickets(maxTicketNumberSaved); // Use the loaded max ticket number

                        generatedTickets.forEach(ticket => {
                            const ticketEl = selectableTicketsContainer.querySelector(`[data-ticket="${ticket.number}"]`);
                            if (ticketEl) {
                                ticketEl.classList.add('selected');
                                // Disable click for all selectable tickets if they are part of the confirmed list
                                ticketEl.style.pointerEvents = 'none';
                            }
                        });

                        // Adjust button states and sections based on loaded data and view mode
                        if (isViewingShared) {
                            // In shared view, disable all modification inputs/buttons
                            maxTicketNumberInput.disabled = true;
                            renderSelectableTicketsBtn.disabled = true;
                            confirmSelectionBtn.disabled = true;
                            pickWinnerBtn.disabled = true; // Winner can only be drawn by owner in private view
                            saveNamesBtn.disabled = true;
                            pixKeyInput.disabled = true;
                            savePixKeyBtn.disabled = true;
                            generateShareLinkBtn.disabled = true; // Cannot generate new share links from a shared view
                            resetBtn.textContent = 'Voltar para Minha Rifa'; // Change reset button text
                            nameAssignmentSection.classList.remove('hidden'); // Show names, but inputs are disabled
                            renderNameAssignmentInputs(); // Render inputs, but they will be disabled
                            nameAssignmentInputsContainer.querySelectorAll('input').forEach(input => input.disabled = true); // Explicitly disable inputs
                            showMessage("Você está visualizando uma rifa compartilhada (somente leitura).", "info");
                        } else {
                            // Private view logic (existing)
                            if (generatedTickets.length > 0) {
                                confirmSelectionBtn.disabled = true;
                                renderSelectableTicketsBtn.disabled = true;
                                pickWinnerBtn.disabled = !currentWinner;
                                nameAssignmentSection.classList.remove('hidden');
                                renderNameAssignmentInputs();
                            } else {
                                confirmSelectionBtn.disabled = false;
                                pickWinnerBtn.disabled = true;
                                nameAssignmentSection.classList.add('hidden');
                            }
                            resetBtn.textContent = 'Reiniciar'; // Reset button text for private view
                            generateShareLinkBtn.disabled = false; // Enable share button in private view
                        }

                    } else {
                        console.log("Nenhum dado de rifa salvo encontrado para esta visualização.");
                        // If no data, initialize UI as if it's the first time
                        resetLocalStateAndUI();
                    }
                }
            } catch (e) {
                console.error("Erro ao carregar dados da rifa: ", e);
                showMessage("Erro ao carregar dados. Por favor, tente novamente.", "error");
                resetLocalStateAndUI(); // Reset UI in case of loading error
            } finally {
                toggleLoading(false);
            }
        }

        /**
         * Deletes all saved raffle data from Firestore.
         * Only deletes private data. Public shares are not deleted by this.
         */
        async function deleteDataFromFirestore() {
            toggleLoading(true);
            try {
                const docRef = getPrivateUserDocRef(); // Always delete private data
                if (docRef) {
                    await deleteDoc(docRef);
                    console.log("Dados da rifa excluídos do Firestore (privado).");
                    showMessage("Dados da rifa reiniciados com sucesso!", "success");
                }
            } catch (e) {
                console.error("Erro ao excluir dados da rifa: ", e);
                showMessage("Erro ao reiniciar dados. Por favor, tente novamente.", "error");
            } finally {
                toggleLoading(false);
            }
        }

        /**
         * Renders the tickets that can be selected by the user.
         * @param {number} maxNumber - The maximum number of tickets to display.
         */
        function renderSelectableTickets(maxNumber) {
            selectableTicketsContainer.innerHTML = ''; // Clear the container
            selectableTickets = []; // Clear the list of selectable tickets
            selectedTicketsForRaffle = []; // Clear locally selected tickets

            if (maxNumber <= 0 || isNaN(maxNumber)) {
                selectableTicketsContainer.innerHTML = '<p class="text-gray-500">Número inválido de bilhetes.</p>';
                return;
            }

            for (let i = 1; i <= maxNumber; i++) {
                const ticketNumber = String(i).padStart(3, '0');
                const ticketSpan = document.createElement('span');
                ticketSpan.textContent = ticketNumber;
                ticketSpan.classList.add('selectable-ticket', 'text-blue-800', 'bg-blue-100', 'text-xs', 'font-semibold', 'mr-2', 'mb-2', 'px-2.5', 'py-0.5', 'rounded-full');
                ticketSpan.dataset.ticket = ticketNumber; // Store the ticket number in the dataset

                ticketSpan.addEventListener('click', () => {
                    // Only allow selection if tickets have not been confirmed for the raffle yet
                    if (confirmSelectionBtn.disabled) {
                        showMessage('Os bilhetes já foram confirmados. Reinicie para fazer novas seleções.', 'info');
                        return;
                    }

                    ticketSpan.classList.toggle('selected');
                    if (ticketSpan.classList.contains('selected')) {
                        selectedTicketsForRaffle.push(ticketNumber);
                    } else {
                        selectedTicketsForRaffle = selectedTicketsForRaffle.filter(t => t !== ticketNumber);
                    }
                    // Sort to maintain visual consistency, but not strictly necessary for logic
                    selectedTicketsForRaffle.sort((a, b) => parseInt(a, 10) - parseInt(b, 10));
                });
                selectableTicketsContainer.appendChild(ticketSpan);
                selectableTickets.push(ticketNumber);
            }
            // Do not display message here, as this function can be called during initial loading.
            // The message will be displayed by loadDataFromFirestore or by clicking the "Generate Options" button.
        }

        /**
         * Renders input fields to assign names to selected tickets.
         */
        function renderNameAssignmentInputs() {
            nameAssignmentInputsContainer.innerHTML = ''; // Clear the container
            if (generatedTickets.length === 0) {
                nameAssignmentInputsContainer.innerHTML = '<p class="text-gray-500">Nenhum bilhete para atribuir nomes.</p>';
                return;
            }

            generatedTickets.forEach(ticket => {
                const div = document.createElement('div');
                div.classList.add('flex', 'items-center', 'mb-2');

                const label = document.createElement('label');
                label.classList.add('text-gray-700', 'text-sm', 'font-medium', 'w-16');
                label.textContent = `Bilhete ${ticket.number}:`;
                div.appendChild(label);

                const input = document.createElement('input');
                input.type = 'text';
                input.classList.add('shadow-sm', 'appearance-none', 'border', 'rounded-lg', 'flex-1', 'py-2', 'px-3', 'text-gray-700', 'leading-tight', 'focus:outline-none', 'focus:ring-2', 'focus:ring-blue-500', 'focus:border-transparent');
                input.placeholder = 'Nome do comprador';
                input.value = ticket.buyer || ''; // Pre-fill with existing name
                input.dataset.ticketNumber = ticket.number; // Associate the input with the ticket number
                div.appendChild(input);

                nameAssignmentInputsContainer.appendChild(div);
            });
        }

        /**
         * Saves the names entered in the input fields to the generatedTickets variable.
         */
        async function saveNames() {
            const nameInputs = nameAssignmentInputsContainer.querySelectorAll('input[type="text"]');
            nameInputs.forEach(input => {
                const ticketNumber = input.dataset.ticketNumber;
                const buyerName = input.value.trim();
                const ticketIndex = generatedTickets.findIndex(t => t.number === ticketNumber);
                if (ticketIndex !== -1) {
                    generatedTickets[ticketIndex].buyer = buyerName;
                }
            });
            await saveDataToFirestore(); // Save updated names to Firestore
            displayConfirmedTickets(generatedTickets); // Update ticket display
            showMessage('Nomes salvos com sucesso!', "success");
        }

        /**
         * Saves the Pix key entered in the input field.
         */
        async function savePixKey() {
            pixKey = pixKeyInput.value.trim();
            displayPixKey.textContent = pixKey || 'Nenhuma chave Pix salva ainda.';
            await saveDataToFirestore(); // Save updated Pix key to Firestore
            showMessage('Chave Pix salva com sucesso!', "success");
        }

        /**
         * Displays the confirmed raffle tickets in the UI, including the buyer's name.
         * @param {Array<Object>} tickets - The array of ticket objects to display.
         */
        function displayConfirmedTickets(tickets) {
            if (tickets.length === 0) {
                ticketsDisplay.innerHTML = '<p class="text-gray-500">Nenhum bilhete confirmado para a rifa ainda.</p>';
            } else {
                ticketsDisplay.innerHTML = tickets.map(ticket => {
                    const isWinner = (currentWinner && ticket.number === currentWinner.number) ? 'winning-ticket' : '';
                    const buyerInfo = ticket.buyer ? ` (${ticket.buyer})` : '';
                    return `<span class="inline-block bg-blue-100 text-blue-800 text-xs font-semibold mr-2 mb-2 px-2.5 py-0.5 rounded-full ${isWinner}">${ticket.number}${buyerInfo}</span>`;
                }).join('');
            }
        }

        /**
         * Selects a random winner from the generated (confirmed) tickets.
         * @returns {Object | null} - The winning ticket object ({ number, buyer }) or null if no tickets.
         */
        function pickWinner() {
            if (generatedTickets.length === 0) {
                showMessage('Por favor, confirme os bilhetes para a rifa primeiro.', "warning");
                return null;
            }
            const randomIndex = Math.floor(Math.random() * generatedTickets.length);
            return generatedTickets[randomIndex];
        }

        /**
         * Clears the local application state and UI, without interacting with Firestore.
         * Used for initialization or after a loading error.
         */
        function resetLocalStateAndUI() {
            selectableTickets = [];
            selectedTicketsForRaffle = [];
            generatedTickets = [];
            currentWinner = null;
            pixKey = '';
            maxTicketNumberSaved = 50; // Reset max ticket number to default
            isViewingShared = false;
            currentShareId = null;

            maxTicketNumberInput.value = maxTicketNumberSaved;
            renderSelectableTickets(maxTicketNumberSaved);

            displayConfirmedTickets([]);
            winnerDisplay.innerHTML = '<span class="text-gray-400 text-lg">Ainda não sorteado</span>';
            nameAssignmentInputsContainer.innerHTML = '<p class="text-gray-500">Selecione e confirme os bilhetes para atribuir nomes.</p>';
            nameAssignmentSection.classList.add('hidden');

            pixKeyInput.value = '';
            displayPixKey.textContent = 'Nenhuma chave Pix salva ainda.';

            // Re-enable all inputs and buttons for private view
            maxTicketNumberInput.disabled = false;
            renderSelectableTicketsBtn.disabled = false;
            confirmSelectionBtn.disabled = false;
            pickWinnerBtn.disabled = true;
            saveNamesBtn.disabled = false;
            pixKeyInput.disabled = false;
            savePixKeyBtn.disabled = false;
            generateShareLinkBtn.disabled = false; // Enable share button
            shareLinkDisplaySection.classList.add('hidden'); // Hide share link display
            shareLinkDisplay.value = ''; // Clear share link input

            resetBtn.textContent = 'Reiniciar'; // Reset button text for private view

            const selectableTicketElements = selectableTicketsContainer.querySelectorAll('.selectable-ticket');
            selectableTicketElements.forEach(ticketEl => {
                ticketEl.classList.remove('selected');
                ticketEl.style.pointerEvents = 'auto';
            });
            nameAssignmentInputsContainer.querySelectorAll('input').forEach(input => input.disabled = false); // Re-enable name inputs

            messageBox.classList.add('hidden');
            showMessage('Aplicação reiniciada. Gere as opções de bilhetes para começar.', "info");
        }

        /**
         * Resets the application. If viewing a shared link, it navigates back to the private view.
         * Otherwise, it deletes private data and resets locally.
         */
        async function resetApp() {
            if (isViewingShared) {
                window.location.href = window.location.origin + window.location.pathname; // Navigate back to private view
            } else {
                await deleteDataFromFirestore(); // Only delete private data
                resetLocalStateAndUI();
            }
        }

        /**
         * Generates a shareable link for the current raffle data.
         */
        async function generateShareLink() {
            if (generatedTickets.length === 0) {
                showMessage("Por favor, confirme os bilhetes e nomes antes de gerar um link de compartilhamento.", "warning");
                return;
            }

            toggleLoading(true);
            try {
                const shareId = crypto.randomUUID(); // Generate a unique ID for the share
                await saveDataToFirestore(true, shareId); // Save current data to a public share document

                const shareUrl = `${window.location.origin}${window.location.pathname}?shareId=${shareId}`;
                shareLinkDisplay.value = shareUrl;
                shareLinkDisplaySection.classList.remove('hidden');
                showMessage("Link de compartilhamento gerado com sucesso!", "success");

            } catch (e) {
                console.error("Erro ao gerar link de compartilhamento:", e);
                showMessage("Erro ao gerar link de compartilhamento. Por favor, tente novamente.", "error");
            } finally {
                toggleLoading(false);
            }
        }

        /**
         * Copies the share link to the clipboard.
         */
        function copyShareLink() {
            const shareLinkInput = document.getElementById('shareLinkDisplay');
            shareLinkInput.select(); // Select the text in the input field
            shareLinkInput.setSelectionRange(0, 99999); // For mobile devices

            let copiedSuccessfully = false;
            try {
                copiedSuccessfully = document.execCommand('copy');
            } catch (err) {
                console.error('Erro ao tentar copiar o link:', err);
                copiedSuccessfully = false;
            }

            if (copiedSuccessfully) {
                showMessage("Link copiado para a área de transferência!", "success");
                // Optional: Add a temporary visual feedback to the input field
                shareLinkInput.classList.add('border-green-500', 'ring-2', 'ring-green-500');
                setTimeout(() => {
                    shareLinkInput.classList.remove('border-green-500', 'ring-2', 'ring-green-500');
                }, 1500);
            } else {
                // Fallback for cases where direct copy fails
                showMessage("Não foi possível copiar automaticamente. Por favor, selecione o texto no campo acima e copie manualmente (Ctrl+C ou Cmd+C).", "warning");
                // Highlight the input field more prominently for manual copy
                shareLinkInput.focus();
                shareLinkInput.classList.add('border-red-500', 'ring-2', 'ring-red-500');
                setTimeout(() => {
                    shareLinkInput.classList.remove('border-red-500', 'ring-2', 'ring-red-500');
                }, 3000);
            }
        }

        // Event Listeners

        // Button to generate clickable ticket options
        renderSelectableTicketsBtn.addEventListener('click', async () => {
            const maxNum = parseInt(maxTicketNumberInput.value, 10);
            if (isNaN(maxNum) || maxNum <= 0) {
                showMessage('Por favor, insira um número válido de bilhetes disponíveis (maior que zero).', "error");
                return;
            }
            maxTicketNumberSaved = maxNum; // Update the saved max ticket number
            renderSelectableTickets(maxNum);
            confirmSelectionBtn.disabled = false; // Enable confirm selection button
            pickWinnerBtn.disabled = true; // Disable draw while no tickets are confirmed
            winnerDisplay.innerHTML = '<span class="text-gray-400 text-lg">Ainda não sorteado</span>'; // Clear winner
            nameAssignmentSection.classList.add('hidden'); // Hide name section
            shareLinkDisplaySection.classList.add('hidden'); // Hide share link display
            shareLinkDisplay.value = ''; // Clear share link input
            showMessage(`Foram geradas ${maxNum} opções de bilhetes. Agora, selecione os que deseja incluir na rifa.`, "info");
            await saveDataToFirestore(); // Save the updated max ticket number
        });

        // Button to confirm ticket selection for the raffle
        confirmSelectionBtn.addEventListener('click', async () => {
            if (selectedTicketsForRaffle.length === 0) {
                showMessage('Por favor, selecione pelo menos um bilhete para a rifa.', "warning");
                return;
            }
            // Convert selected numbers into ticket objects with empty names
            generatedTickets = selectedTicketsForRaffle.map(number => ({ number: number, buyer: '' }));
            currentWinner = null; // Clear previous winner
            displayConfirmedTickets(generatedTickets); // Display confirmed tickets
            pickWinnerBtn.disabled = false; // Enable draw button
            confirmSelectionBtn.disabled = true; // Disable after confirming
            renderSelectableTicketsBtn.disabled = true; // Disable generating new options

            // Disable clicks on selectable tickets after confirmation
            const selectableTicketElements = selectableTicketsContainer.querySelectorAll('.selectable-ticket');
            selectableTicketElements.forEach(ticketEl => {
                ticketEl.style.pointerEvents = 'none'; // Disable click for all selectable tickets
            });

            nameAssignmentSection.classList.remove('hidden'); // Show name section
            renderNameAssignmentInputs(); // Render name inputs
            shareLinkDisplaySection.classList.add('hidden'); // Hide share link display
            shareLinkDisplay.value = ''; // Clear share link input

            await saveDataToFirestore(); // Save tickets to Firestore
            showMessage(`${generatedTickets.length} bilhetes foram confirmados para a rifa! Agora, atribua os nomes.`, "success");
        });

        // Button to save names
        saveNamesBtn.addEventListener('click', saveNames);

        // Button to save Pix key
        savePixKeyBtn.addEventListener('click', savePixKey);

        // Button to generate share link
        generateShareLinkBtn.addEventListener('click', generateShareLink);

        // Button to copy share link
        copyShareLinkBtn.addEventListener('click', copyShareLink);

        pickWinnerBtn.addEventListener('click', async () => {
            const winner = pickWinner();
            if (winner) {
                currentWinner = winner; // Store the winner
                winnerDisplay.textContent = `${currentWinner.number} (${currentWinner.buyer || 'Sem nome'})`;
                displayConfirmedTickets(generatedTickets); // Update display to highlight winner
                pickWinnerBtn.disabled = true; // Disable after drawing
                await saveDataToFirestore(); // Save winner to Firestore
                showMessage(`O bilhete vencedor é: ${winner.number} (${winner.buyer || 'Sem nome'})!`, "success");
            }
        });

        resetBtn.addEventListener('click', resetApp);

        // Initialize the application when the page loads
        document.addEventListener('firebaseAuthReady', () => {
            displayUserId.textContent = window.userId; // Display user ID
            loadDataFromFirestore(); // Load all saved data
        });

        // Ensure the UI is initialized with a default state while waiting for authentication and loading
        window.onload = () => {
            // Render selectable ticket options initially with the default value
            renderSelectableTickets(parseInt(maxTicketNumberInput.value, 10));
            // The loading and UI update logic will be handled by 'firebaseAuthReady'
        };
    </script>
</body>
</html>
