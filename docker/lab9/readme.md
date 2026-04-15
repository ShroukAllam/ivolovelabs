# Node.js & MySQL Docker Compose Application

A containerized Node.js application with MySQL database using Docker Compose - Lab 9 Project.

---

## Technologies Used

- **Node.js 18** - Application runtime
- **MySQL 8.0** - Database
- **Docker & Docker Compose** - Container orchestration

---

##  How to Run the Application

### 1. Clone the repository
```bash
git clone https://github.com/shroukallam/kubernets-app.git
cd kubernets-app


docker-compose up -d --build

http://localhost:3000

<img width="862" height="720" alt="Screenshot 2026-04-16 010520" src="https://github.com/user-attachments/assets/d7fa47ea-710d-472d-91f6-487b04747e99" />

http://localhost:3000/health
<img width="840" height="95" alt="health" src="https://github.com/user-attachments/assets/3c1460c0-ccf4-4ac7-a422-251a729472c0" />

<img width="626" height="210" alt="Screenshot 2026-04-16 011856" src="https://github.com/user-attachments/assets/07a9ca60-5cd6-430f-b04b-e09f1f023193" />


http://localhost:3000/ready



