A loyalty management system for cafés. Staff can search members by phone number, record purchases, automatically add loyalty points, and redeem rewards. The system keeps the points balance accurate and maintains a complete transaction history.

Tech Stack
Frontend: React, TypeScript, Vite, Tailwind CSS
Backend: Node.js, Express, TypeScript
Database: MongoDB with Mongoose
Authentication: JWT + bcrypt
Testing: Vitest + Supertest
Project Structure
cafe-rewards/
├── client/       # React frontend
├── server/       # Node.js/Express backend
├── README.md
├── REASONING.md
└── AI_LOGS.md
Setup
1. Requirements

Install:

Node.js 18+
npm
MongoDB
2. Install Dependencies

From the project root:

npm install
3. Configure Environment

Create a .env file in the root directory:

PORT=5000
MONGODB_URI=mongodb://127.0.0.1:27017/cafe_rewards
JWT_SECRET=your_secret_key
CLIENT_URL=http://localhost:5173
VITE_API_URL=http://localhost:5000/api
NODE_ENV=development

Make sure MongoDB is running before starting the application.

4. Add Sample Data

Run:

npm run seed

This adds sample staff accounts, members, rewards, purchases and transaction history.

Run the Project
Start Frontend and Backend
npm run dev

Frontend:

http://localhost:5173

Backend:

http://localhost:5000
Run Separately

Backend:

npm run dev --workspace=server

Frontend:

npm run dev --workspace=client
Test the Application

Run all automated tests:

npm run test

The tests cover points calculation, tier upgrades, redemption, insufficient points, duplicate phone numbers, balance consistency and concurrent redemptions. The project verification recorded 15/15 tests passing.

Debugging
Backend not starting

Check Node and npm:

node -v
npm -v

Then reinstall dependencies:

npm install
MongoDB connection error

Check that MongoDB is running and verify:

MONGODB_URI=mongodb://127.0.0.1:27017/cafe_rewards
Frontend cannot connect to backend

Make sure the backend is running on:

http://localhost:5000

and check:

VITE_API_URL=http://localhost:5000/api
Check Backend Health

Open:

http://localhost:5000/api/health

A healthy response should indicate that the backend is running.

Points or balance problem

Check the member's transaction history:

GET /api/transactions/member/:memberId

Admins can also check the balance against the ledger:

GET /api/admin/reconciliation/member/:id
API Endpoints

Base URL:

http://localhost:5000/api
