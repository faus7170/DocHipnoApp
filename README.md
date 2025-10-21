# 🧠 HipnoApp – API Technical Documentation

> **Internal Use Only – HipnoApp Development Department**

This repository contains the technical documentation and resources required for consuming, testing, and maintaining the **HipnoApp Backend API**.  
It includes Postman collections, OpenAPI (Swagger) specifications, and detailed technical documentation of all endpoints.

---

## 🗂️ Repository Structure

| File | Description |
|------|--------------|
| **HipnoAppV1.postman_collection.json** | Postman collection containing all backend API requests. |
| **stagin.postman_environment.json** | Postman environment file configured for the *staging* environment. |
| **openapi_all.yaml** | **OpenAPI 3.0** specification used for Swagger documentation or external integrations. |
| **HipnoApp_Backend_API.md** | Technical document detailing each endpoint, authentication, parameters, and response examples. |

---

## 🚀 Usage Guide

### 1. Import the Postman Collection

1. Open **Postman**.  
2. Go to **Import → Files** and select `HipnoAppV1.postman_collection.json`.  
3. Import the `stagin.postman_environment.json` file as well.  
4. Choose the “staging” environment before executing any requests.

> ⚙️ Make sure to configure the correct access tokens or credentials before sending requests.

---

### 2. View the API with Swagger

To explore the API visually:

1. Open [Swagger Editor](https://editor.swagger.io/).  
2. Load the `openapi_all.yaml` file.  
3. You can browse through endpoints, inspect parameters and responses, and test requests directly in the browser.

---

### 3. Review the Technical Documentation

Check [`HipnoApp_Backend_API.md`](./HipnoApp_Backend_API.md) for:

- General description of the backend architecture.  
- Detailed module and endpoint documentation.  
- Authentication and role-based access rules.  
- Request/response examples.  
- Security and best-practice notes.

---

## 🌍 Environments

| Environment | Base URL | Description |
|--------------|-----------|-------------|
| **Staging** | *(Insert current staging URL)* | Internal testing environment. |
| **Production** | *(Insert production URL)* | Live operational environment. |

> 🔒 **Note:** Access credentials for each environment are managed by the Backend Team. Do not share externally.

---

## 🧩 Recommended Tools

- [Postman](https://www.postman.com/downloads/)  
- [Swagger Editor](https://editor.swagger.io/)  
- [Node.js](https://nodejs.org/) (for API utilities if needed)  
- [VS Code](https://code.visualstudio.com/) (for YAML and Markdown editing)

---

## 🧾 Version Control

| Version | Date | Description |
|----------|------|-------------|
| **v1.0** | September 2025 | Initial version of HipnoApp API documentation. |
| **v1.1** | October 2025 | Environment updates and documentation normalization. |

---

## 👥 Technical Contact

**Backend Developer:**  
👨‍💻 *Fausto Lamiña*  
📧 [fausto.laminia@gmail.com](mailto:fausto.laminia@gmail.com)

For internal support or integration inquiries, please contact the **HipnoApp Backend Team**.

---

## 🔒 Confidentiality Notice

This repository contains internal technical information.  
Unauthorized distribution or publication outside the authorized HipnoApp team is **strictly prohibited**.

---

© 2025 **HipnoApp** – All rights reserved.
