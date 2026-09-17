# CST8915 Lab 1: Algonquin Pet Store on Azure VM

**Student Name**: Peng Wang
**Student ID**: 041107730
**Course**: CST8915 Full-stack Cloud-native Development
**Semester**: Fall 2026

---

## Demo Video

🎥 [Watch Demo Video](https://www.youtube.com/watch?v=YOUR_VIDEO_ID)

---

## Technical Explanations

### Order Service (Node.js)

The Order Service accepts customer orders and hands them to RabbitMQ. It is a small Express application (`index.js`) with a single route, `POST /orders`, listening on port 3000. Node.js with Express fits this job because the service does almost no computation: it parses a JSON body and waits on network I/O, which is exactly what Node's event loop handles well, and the `amqplib` package gives it a ready-made AMQP 0-9-1 client. The `cors` middleware is enabled for every route because the browser loads the page from port 8080 and then calls port 3000, which the browser treats as a different origin.

In the architecture it is the **producer**. For each request it opens a connection to `amqp://localhost`, creates a *confirm channel*, asserts the durable queue `order_queue`, and publishes the order as a persistent, mandatory message. It replies `Order received` only after RabbitMQ confirms the message, and returns HTTP 500 if the connection, channel, or routing fails; the `finish()` helper makes sure exactly one response is sent and the connection is closed. The service never talks to the Product Service and does not know who will consume the order, so order intake keeps working even though this lab has no consumer: messages simply stay in the queue. One gap against the Twelve-Factor "Config" factor is that the RabbitMQ URL and the port are constants in the code instead of environment variables.

### Product Service (Rust)

The Product Service owns the product catalog. It is a Rust program (`src/main.rs`) built on the Warp web framework and the Tokio async runtime, with `serde_json` producing the JSON. It exposes one read-only endpoint, `GET /products`, on `0.0.0.0:3030`, and returns three hard-coded products: Dog Food $19.99, Cat Food $34.99, and Bird Seeds $10.99. Rust compiles to a single native binary with no garbage collector, so a catalog endpoint that every page load hits stays fast and memory-safe with a very small footprint. The trade-off is visible in the lab: the first `cargo run` has to compile all dependencies, which takes several minutes on a B2s VM.

In the architecture it is an independent, stateless service: it has no database, does not use RabbitMQ, and never calls another service, which is why it can be installed and started before or after the Order Service. Its only client is the Store Front, which calls it over HTTP/REST from the browser. Because that call is cross-origin, the route is wrapped in a Warp CORS filter that allows any origin but only the `GET` method. Binding to `0.0.0.0` rather than `127.0.0.1` is what lets the request from my laptop reach it through the VM's public IP once the NSG rule for port 3030 exists.

### Store Front (Vue.js)

The Store Front is the customer-facing single-page application, built with Vue 3 and served in this lab by the Vue CLI development server on port 8080. All the logic is in `src/components/OrderForm.vue`: the `created()` hook fetches the catalog, radio buttons bind the chosen product through `v-model`, a computed property `totalPrice` multiplies price by quantity (two Dog Food gives $39.98), and the **Place Order** button stays disabled until a product and a positive quantity are selected. Vue suits this because its reactive data binding updates the total and the button state without any manual DOM code.

In the architecture it is the only part the user sees, and it holds no business data of its own. It talks to both back ends with the browser's Fetch API: `GET http://<VM-IP>:3030/products` to the Product Service, and `POST http://<VM-IP>:3000/orders` with a JSON body of `product`, `quantity`, and `totalPrice` to the Order Service. The key detail is that these requests are sent by the browser on my laptop, not by the VM, so `localhost` in the original code had to be replaced with the VM's public IP, and ports 3000 and 3030 had to be opened in the NSG alongside 8080. It never touches RabbitMQ directly; it only learns that the order was queued from the Order Service's HTTP response.

---

## Challenges and Learnings (Optional)

[To fill in after the deployment]

---

## Acknowledgments

- Lab instructions and source code: `ramymohamed10/26F_Lab1_CST8915`.
- GenAI declaration: I used Claude (Anthropic) to help me read the service source code and to review the wording of this README, as permitted for labs in this course. The deployment and the demo video are my own work.
