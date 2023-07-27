# websocket-promise
How to use a WebSocket connection for one-time communication

## Sample Code

```javascript
/**
 * Creates a WebSocket connection capable
 * of handling one-off messages as a promise.
 * Note: This isn't a true proxy of the WebSocket connection interface
 */
class WSProxy {
  #url = "";
  #connection = null;
  #promises = new Map();
  #messageHandler = console.log;

  constructor(url) {
    this.#url = url;
  }

  get onmessage() {
    return this.#connection.onmessage;
  }

  set onmessage(fn) {
    if (typeof fn === "function") {
      this.#messageHandler = fn;
    }
  }

  #handleOpen = (resolve) => (event) => {
    resolve(event);
  };

  connect = () => {
    this.#connection = new WebSocket(this.#url);
    this.#connection.onmessage = this.#handleResponse;
    return new Promise((resolve) => {
      this.#connection.onopen = this.#handleOpen(resolve);
    });
  };

  #handleResponse = (event) => {
    const data = event.data;
    const statusCode = data?.statusCode ?? 400;
    const hasPromise = this.#promises.has(data.id);
    const isError = statusCode >= 400;

    if (hasPromise && isError) {
      const { reject } = this.#promises.get(data.id);
      reject(new Error(data.body));
      return this.#promises.delete(data.id);
    }

    if (hasPromise) {
      const { resolve } = this.#promises.get(data.id);
      resolve(event);
      return this.#promises.delete(data.id);
    }

    this.#messageHandler(event);
  };

  postMessage = (id, args) => {
    return new Promise((resolve, reject) => {
      this.#promises.set(id, { resolve, reject });
      this.#connection.send(JSON.stringify({ id, args }));
    });
  };

  send = (...args) => this.#connection.send(...args);
}

async function startChatSession(connection) {
  try {
    await connection.connect();
    const sessionMeta = await connection.postMessage("1", {
      name: "Ezell Frazier"
    });

    return sessionMeta;
  } catch (error) {
    console.error("Failed to initialize session", error);
  }
}

async function chatApp() {
  const chatLog = [];
  const connection = new WSProxy("ws://localhost:3000");
  const sessionMeta = await startChatSession(connection);

  connection.onmessage = (event) => {
    chatLog.push(event.data)
    console.log('chat log', chatLog)
  };

  const submitBtn = document.getElementById("submit");
  const messageInput = document.getElementById("messageInput");
  const message = messageInput.value;

  submitBtn.onclick(() => {
    chatLog.push(message);
    connection.send(
      JSON.stringify({
        sessionMeta,
        message
      })
    );
  });
}

chatApp()

```
