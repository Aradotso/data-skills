---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket routing, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "configure JWT authentication for user system"
  - "integrate AI ticket classification"
  - "build admin panel with user management"
  - "deploy user management system with ML service"
  - "implement burnout detection and anomaly alerts"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a full-stack enterprise user management system featuring role-based access control, task tracking with Kanban boards, support ticket management, and AI-powered analytics including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

## What It Does

- **User Management**: Secure authentication, role-based access (Admin/User), user CRUD operations
- **Task Management**: Kanban boards (To Do → In Progress → Done), time tracking, task assignment
- **Support Tickets**: Create, track, and manage tickets with AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, suspicious activity alerts
- **User Dashboard**: Personal task overview, performance insights, notifications

## Architecture

The system consists of three main components:
- **Frontend**: React.js application (port 3000)
- **Backend**: Node.js REST API with JWT authentication (port 5000)
- **ML Service**: FastAPI with scikit-learn and River for online learning (port 8000)
- **Database**: MongoDB for persistent storage

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB (local or Atlas)
```

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Key API Endpoints

### Authentication

```javascript
// Register user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer <token>" }

// Update user
PUT /api/users/:userId
Headers: { Authorization: "Bearer <token>" }
{
  "name": "Updated Name",
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:userId
Headers: { Authorization: "Bearer <token>" }
```

### Task Management

```javascript
// Create task
POST /api/tasks
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Implement feature X",
  "description": "Add new functionality",
  "assignedTo": "userId",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PUT /api/tasks/:taskId
{
  "status": "in-progress"
}

// Get user tasks
GET /api/tasks/user/:userId
Headers: { Authorization: "Bearer <token>" }
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { Authorization: "Bearer <token>" }
{
  "subject": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// AI-powered ticket classification
POST /api/ml/classify-ticket
{
  "subject": "Password reset not working",
  "description": "I tried resetting my password but didn't receive email"
}
// Returns: { category: "technical", priority: "medium", suggestedAssignee: "supportTeamId" }
```

### AI Analytics Endpoints

```javascript
// Risk prediction
POST /api/ml/predict-risk
{
  "userId": "user123",
  "taskCount": 15,
  "overdueCount": 3,
  "avgCompletionTime": 48,
  "ticketCount": 2
}
// Returns: { riskLevel: "medium", score: 0.65, factors: [...] }

// Anomaly detection
POST /api/ml/detect-anomaly
{
  "userId": "user123",
  "loginTime": "03:00",
  "ipAddress": "192.168.1.100",
  "accessPattern": "unusual"
}
// Returns: { isAnomaly: true, confidence: 0.87, reason: "Unusual login time" }

// Burnout analysis
POST /api/ml/analyze-burnout
{
  "userId": "user123",
  "workHours": 65,
  "taskLoad": 25,
  "overtimeDays": 12,
  "weekendWork": true
}
// Returns: { burnoutRisk: "high", score: 0.82, recommendations: [...] }

// Project delay prediction
POST /api/ml/predict-delay
{
  "projectId": "proj123",
  "tasksCompleted": 45,
  "tasksRemaining": 30,
  "daysLeft": 14,
  "teamVelocity": 2.5
}
// Returns: { delayProbability: 0.73, estimatedDelay: 5, suggestions: [...] }
```

## Frontend Usage Patterns

### Authentication Setup

```javascript
// src/utils/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const api = axios.create({
  baseURL: API_URL,
});

// Add JWT token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default api;
```

### Login Component

```javascript
// src/components/Login.js
import React, { useState } from 'react';
import api from '../utils/api';

const Login = () => {
  const [credentials, setCredentials] = useState({ email: '', password: '' });
  const [error, setError] = useState('');

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const response = await api.post('/api/auth/login', credentials);
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
      window.location.href = response.data.user.role === 'admin' 
        ? '/admin/dashboard' 
        : '/user/dashboard';
    } catch (err) {
      setError(err.response?.data?.message || 'Login failed');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        placeholder="Email"
        value={credentials.email}
        onChange={(e) => setCredentials({...credentials, email: e.target.value})}
        required
      />
      <input
        type="password"
        placeholder="Password"
        value={credentials.password}
        onChange={(e) => setCredentials({...credentials, password: e.target.value})}
        required
      />
      {error && <div className="error">{error}</div>}
      <button type="submit">Login</button>
    </form>
  );
};

export default Login;
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import api from '../utils/api';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const user = JSON.parse(localStorage.getItem('user'));
    const response = await api.get(`/api/tasks/user/${user._id}`);
    
    const grouped = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await api.put(`/api/tasks/${taskId}`, { status: newStatus });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(column => (
        <div key={column} className="kanban-column">
          <h3>{column === 'inProgress' ? 'In Progress' : column.toUpperCase()}</h3>
          {tasks[column].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status} 
                onChange={(e) => moveTask(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in-progress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import api from '../utils/api';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Get user stats
      const userStats = await api.get(`/api/users/${userId}/stats`);
      
      // Get AI predictions
      const [riskData, burnoutData] = await Promise.all([
        api.post('/api/ml/predict-risk', {
          userId,
          taskCount: userStats.data.taskCount,
          overdueCount: userStats.data.overdueCount,
          avgCompletionTime: userStats.data.avgCompletionTime,
          ticketCount: userStats.data.ticketCount
        }),
        api.post('/api/ml/analyze-burnout', {
          userId,
          workHours: userStats.data.weeklyHours,
          taskLoad: userStats.data.taskCount,
          overtimeDays: userStats.data.overtimeDays,
          weekendWork: userStats.data.weekendWork
        })
      ]);

      setAnalytics({
        risk: riskData.data,
        burnout: burnoutData.data,
        stats: userStats.data
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="analytics-card">
        <h3>Risk Assessment</h3>
        <div className={`risk-level ${analytics.risk.riskLevel}`}>
          {analytics.risk.riskLevel.toUpperCase()}
        </div>
        <p>Score: {(analytics.risk.score * 100).toFixed(1)}%</p>
        <ul>
          {analytics.risk.factors.map((factor, i) => (
            <li key={i}>{factor}</li>
          ))}
        </ul>
      </div>

      <div className="analytics-card">
        <h3>Burnout Analysis</h3>
        <div className={`burnout-level ${analytics.burnout.burnoutRisk}`}>
          {analytics.burnout.burnoutRisk.toUpperCase()} RISK
        </div>
        <p>Score: {(analytics.burnout.score * 100).toFixed(1)}%</p>
        <ul>
          {analytics.burnout.recommendations.map((rec, i) => (
            <li key={i}>{rec}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Backend Implementation Patterns

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user) {
      return res.status(401).json({ message: 'User not found' });
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const isAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authenticate, isAdmin };
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

const createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      createdBy: req.user._id,
      priority,
      dueDate,
      status: 'todo'
    });

    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

const updateTaskStatus = async (req, res) => {
  try {
    const { taskId } = req.params;
    const { status } = req.body;

    const task = await Task.findById(taskId);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    // Check if user owns the task or is admin
    if (task.assignedTo.toString() !== req.user._id.toString() && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Not authorized' });
    }

    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }

    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

const getUserTasks = async (req, res) => {
  try {
    const { userId } = req.params;
    const tasks = await Task.find({ assignedTo: userId })
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

module.exports = { createTask, updateTaskStatus, getUserTasks };
```

## ML Service Implementation

### FastAPI Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Dict
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models (or initialize)
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

class RiskPredictionRequest(BaseModel):
    userId: str
    taskCount: int
    overdueCount: int
    avgCompletionTime: float
    ticketCount: int

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    workHours: float
    taskLoad: int
    overtimeDays: int
    weekendWork: bool

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Calculate risk score based on metrics
        features = np.array([
            request.taskCount,
            request.overdueCount,
            request.avgCompletionTime,
            request.ticketCount
        ]).reshape(1, -1)
        
        # Simple heuristic (replace with trained model)
        overdue_ratio = request.overdueCount / max(request.taskCount, 1)
        time_score = min(request.avgCompletionTime / 72, 1)  # 72h baseline
        
        risk_score = (overdue_ratio * 0.4) + (time_score * 0.3) + (min(request.ticketCount / 10, 1) * 0.3)
        
        if risk_score > 0.7:
            risk_level = "high"
        elif risk_score > 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        factors = []
        if overdue_ratio > 0.2:
            factors.append(f"High overdue rate: {overdue_ratio*100:.1f}%")
        if request.avgCompletionTime > 48:
            factors.append(f"Slow completion time: {request.avgCompletionTime:.1f}h avg")
        if request.ticketCount > 5:
            factors.append(f"Many support tickets: {request.ticketCount}")
        
        return {
            "riskLevel": risk_level,
            "score": float(risk_score),
            "factors": factors
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    try:
        # Burnout score calculation
        hours_score = min(request.workHours / 60, 1)  # 60h+ is max
        task_score = min(request.taskLoad / 30, 1)  # 30+ tasks is max
        overtime_score = min(request.overtimeDays / 20, 1)  # 20+ days is max
        weekend_score = 0.3 if request.weekendWork else 0
        
        burnout_score = (hours_score * 0.35) + (task_score * 0.25) + (overtime_score * 0.25) + weekend_score
        
        if burnout_score > 0.7:
            risk = "high"
        elif burnout_score > 0.4:
            risk = "medium"
        else:
            risk = "low"
        
        recommendations = []
        if request.workHours > 50:
            recommendations.append("Reduce weekly work hours to under 45")
        if request.taskLoad > 20:
            recommendations.append("Redistribute tasks to balance workload")
        if request.overtimeDays > 10:
            recommendations.append("Limit overtime to essential tasks only")
        if request.weekendWork:
            recommendations.append("Avoid weekend work for better work-life balance")
        
        return {
            "burnoutRisk": risk,
            "score": float(burnout_score),
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        text = f"{request.subject} {request.description}".lower()
        
        # Simple keyword-based classification (replace with NLP model)
        if any(word in text for word in ['login', 'password', 'access', 'authentication']):
            category = "technical"
            priority = "high"
        elif any(word in text for word in ['bug', 'error', 'crash', 'not working']):
            category = "technical"
            priority = "high"
        elif any(word in text for word in ['feature', 'request', 'suggest', 'improvement']):
            category = "feature_request"
            priority = "low"
        elif any(word in text for word in ['account', 'billing', 'payment']):
            category = "account"
            priority = "medium"
        else:
            category = "general"
            priority = "medium"
        
        return {
            "category": category,
            "priority": priority,
            "suggestedAssignee": "support-team"  # Could be smarter routing
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=<generate-secure-random-string>
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=production
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**
```bash
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
PYTHONUNBUFFERED=1
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Deployment

### Vercel (Frontend)

```bash
cd frontend
npm run build

# Install Vercel CLI
npm i -g vercel

# Deploy
vercel --prod
```

### Heroku (Backend)

```bash
# Create Procfile
echo "web: node server.js" > Procfile

# Deploy
heroku create your-app-name
heroku config:set JWT_SECRET=your_secret
heroku config:set MONGODB_URI=your_mongodb_atlas_uri
git push heroku main
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  mongodb:
    image: mongo:5
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/enterprise-user-mgmt
      - JWT_SECRET=${JWT_SECRET}
      - ML_SERVICE_URL=http://ml-service:8000
    depends_on:
      - mongodb

  ml-service:
    build: ./ml-service
    ports:
      - "8000:8000"
    environment:
      - BACKEND_URL=http://backend:5000
    volumes:
      - ./ml-service/models:/app/models

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:5000
      - REACT_APP_ML_API_URL=http://localhost:8000

volumes:
  mongo-data:
```

## Troubleshooting

### CORS Issues
```javascript
// backend/server.js
const cors = require('cors');
app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### JWT Token Expiration
```javascript
// frontend/src/utils/api.js
api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### MongoDB Connection Issues
```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Service Not Responding
```bash
# Check ML service logs
cd ml-service
uvicorn main:app --reload --log-level debug

# Test endpoint directly
curl http://localhost:8000/health
```

### Task Status Not Updating
```javascript
// Ensure proper task update logic
const updateTask = async (taskId, updates) => {
  const response = await api.put(`/api/tasks/${taskId}`, updates);
  // Force re-fetch to sync state
  await fetchTasks();
  return response.data;
};
```
