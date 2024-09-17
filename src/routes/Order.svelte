<script>
    import axios from 'axios';

    let products = [];
    let selectedItems = [];
    let tableId = ""; // Número de mesa ingresado por el usuario
    let printers = []; // Lista de impresoras
    let selectedKitchenPrinter = ""; // Impresora seleccionada para la cocina
    let selectedBarPrinter = ""; // Impresora seleccionada para el bar
    let comment = ""; // Comentario adicional para el pedido
    let messageResolver;

    // Cargar productos desde el backend
    const loadProducts = async () => {
        try {
            const response = await axios.get('http://192.168.1.35:3000/products');
            products = response.data;
            console.log(products);
        } catch (error) {
            console.error('Error loading products:', error);
        }
    };

    // Detectar si estamos en un entorno WebView
    if (typeof window !== 'undefined' && window.chrome && window.chrome.webview) {
        console.log("Entorno WebView detectado");
        window.chrome.webview.addEventListener('message', (event) => {
            const messageString = event.data;
            console.log("Mensaje recibido:", messageString);

            let messageData;
            try {
                messageData = JSON.parse(messageString);
            } catch (error) {
                console.error("Error al parsear el mensaje:", error);
                return;
            }

            // Verificar la acción del mensaje recibido
            /*if (messageData.action === "listPrinters") {
                printers = messageData.printers;
                console.log("Impresoras recibidas:", printers);
                if (messageResolver) {
                    messageResolver(printers);
                }
            } else*/ if (messageData.action === "printerstatus") {
                // Manejar los estados de la impresora y mostrar alertas
                alert(`${messageData.code}: ${messageData.message}`);
            } else {
                console.log("Acción no reconocida:", messageData.action);
                if (messageResolver) {
                    messageResolver(null);
                }
            }
        });
    } else {
        console.log("No se está ejecutando en un entorno de WebView.");
    }

    // Solicitar la lista de impresoras desde el WebView
    const requestPrinters = async () => {
        if (typeof window !== 'undefined' && window.chrome && window.chrome.webview) {
            const data = { action: "getPrinters" };
            console.log("Solicitando lista de impresoras...");
            window.chrome.webview.postMessage(data);

            try {
                const result = await new Promise((resolve) => {
                    messageResolver = resolve;
                    window.chrome.webview.postMessage(JSON.stringify(data));
                });

                if (result) {
                    printers = result;
                    console.log("Impresoras actualizadas:", printers);
                } else {
                    console.log("No se recibieron impresoras");
                }
            } catch (error) {
                console.error("Error al obtener impresoras:", error);
            } finally {
                messageResolver = null;
            }
        } else {
            console.log("No se está ejecutando en un entorno de WebView.");
        }
    };

    // Seleccionar o deseleccionar un producto
    const toggleProduct = (product, quantity) => {
        const index = selectedItems.findIndex(item => item.id === product.id);
        if (index === -1) {
            selectedItems.push({ ...product, quantity: quantity.toString() });
        } else {
            selectedItems[index].quantity = quantity.toString();
        }
    };

    // Crear y enviar el pedido
    const submitOrder = async () => {
        if (tableId.trim() === "" || selectedItems.length === 0) {
            alert("Por favor, ingresa el número de mesa y selecciona al menos un producto.");
            return;
        }

        try {
            const response = await axios.post('http://192.168.1.35:3000/order', {
                tableId,
                items: selectedItems,
                comment
            });

            const orderId = response.data.orderId;

            alert(`Pedido creado con ID: ${response.data.orderId}`);

            // Enviar el pedido al WebView para impresión
            sendOrderToWebView(orderId);

            // Limpiar los ítems seleccionados
            selectedItems = [];
            tableId = "";
            comment = "";
            document.querySelectorAll('input[type="checkbox"]').forEach(checkbox => checkbox.checked = false);
            document.querySelectorAll('input[type="number"]').forEach(input => input.value = "0");
        } catch (error) {
            console.error('Error creando el pedido:', error);
            alert('Hubo un error al crear el pedido');
        }
    };

    // Enviar pedido al WebView
    const sendOrderToWebView = (orderId) => {
        if (!isWinWebview()) {
            console.log("Este código solo se ejecuta en un entorno de WebView.");
            return;
        }

        const kitchenItems = selectedItems.filter(item => item.category === 'kitchen');
        const barItems = selectedItems.filter(item => item.category === 'bar');

        const data = {
            tableId,
            orderId,
            kitchenItems,
            barItems,
            comment,
            /*kitchenPrinter: selectedKitchenPrinter,
            barPrinter: selectedBarPrinter*/
        };

        sendToWinWebview('printOrder', data);
    };

    const sendToWinWebview = (action, data) => {
        if (isWinWebview()) {
            const _configs = JSON.parse(localStorage.getItem("config")) || {};
            data.action = action;
            data.configuration = _configs;

            console.log("Enviando datos al WebView:", data);
            window.chrome.webview.postMessage(data);
        } else {
            console.log("No está en un entorno de WebView.");
        }
    };

    const isWinWebview = () => window['chrome'] && window['chrome']['webview'] ? true : false;

    // Cargar productos al montar el componente
    loadProducts();

</script>

<div class="container">
    <!-- Sección de pedidos -->
    <div class="left-pane">
        <h2>Selecciona tus productos</h2>

        <!-- Campo para ingresar el número de mesa -->
        <div>
            <label for="tableId">Número de Mesa:</label>
            <input type="text" id="tableId" bind:value={tableId} placeholder="Ingresa el número de mesa" />
        </div>

        <!-- Lista de productos -->
        <div>
            {#each products as product}
            <div>
                <input type="checkbox" id={product.id} on:change={(event) => toggleProduct(product, event.target.nextSibling.value)} />
                <label for={product.id}>{product.name} - ${product.price}</label>
                <input type="number" min="1" value="0" placeholder="Cantidad" on:input={(event) => toggleProduct(product, event.target.value)} />
            </div>
            {/each}
        </div>

        <!-- Campo de comentario adicional -->
        <div>
            <label for="comment">Comentario:</label>
            <textarea id="comment" bind:value={comment} placeholder="Agrega un comentario"></textarea>
        </div>

        <button on:click={submitOrder}>Realizar Pedido</button>
    </div>

    <!-- Sección de impresoras -->
    <!--<div class="right-pane">
        <h2>Impresoras Disponibles</h2>
        <button on:click={requestPrinters}>Listar Impresoras</button>

  
        <div>
            <label for="kitchenPrinter">Cocina:</label>
            <select id="kitchenPrinter" bind:value={selectedKitchenPrinter}>
                <option value="" disabled selected>Selecciona una impresora</option>
                {#each printers as printer}
                    <option value={printer}>{printer}</option>
                {/each}
            </select>
        </div>


        <div>
            <label for="barPrinter">Bar:</label>
            <select id="barPrinter" bind:value={selectedBarPrinter}>
                <option value="" disabled selected>Selecciona una impresora</option>
                {#each printers as printer}
                    <option value={printer}>{printer}</option>
                {/each}
            </select>
        </div>
        
    </div>  -->
</div>

<style>
    .container {
        display: flex;
    }

    .left-pane, .right-pane {
        flex: 1;
        padding: 20px;
    }

    .left-pane {
        border-right: 1px solid #ccc;
    }

    .right-pane {
        padding-left: 20px;
    }

    button {
        margin-top: 20px;
        padding: 10px 20px;
        font-size: 16px;
        background-color: #ff3e00;
        color: white;
        border: none;
        cursor: pointer;
    }

    select {
        margin-top: 10px;
        padding: 5px;
        font-size: 16px;
        width: 100%;
    }

    textarea {
        width: 100%;
        margin-top: 10px;
        padding: 10px;
        font-size: 16px;
    }
</style>
