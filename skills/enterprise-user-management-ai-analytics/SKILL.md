---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, anomaly detection, and task optimization
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement user task tracking with AI insights"
  - "configure JWT authentication for enterprise app"
  - "deploy user management system with ML service"
  - "create admin dashboard with AI analytics"
  - "build kanban board with time tracking"
  - "set up FastAPI ML service for user analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication via JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support System**: Ticket creation, AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization-wide analytics, audit logs, alerts

The system consists of three main components:
1. **Frontend** (React.js) - User interface for admins and users
2. **Backend** (Node.js) - REST APIs, authentication, business logic
3. **ML Service** (FastAPI) - AI/ML models for analytics and predictions

## Installation

### Prerequisites

- Node.js (v14+)
- Python 3.8+
- MongoDB

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install
```

Create `.env` file:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
NODE_ENV=development
```

Start backend:

```bash
npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
```

ML service runs at `http://localhost:8000`

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

Frontend runs at `http://localhost:3000`

## Backend API Endpoints

### Authentication

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  const data = await response.json();
  // Store token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

```javascript
// GET /api/users - Get all users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'in-progress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status })
  });
  return response.json();
};
```

### Support Tickets

```javascript
// POST /api/tickets - Create ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets
const getTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

## ML Service API

### Risk Prediction

```python
# POST /api/ml/predict-risk
import requests

def predict_user_risk(user_data, ml_api_url):
    """
    Predict risk score for a user based on behavior
    """
    response = requests.post(
        f"{ml_api_url}/api/ml/predict-risk",
        json={
            "userId": user_data["id"],
            "loginFrequency": user_data["login_count"],
            "taskCompletionRate": user_data["completion_rate"],
            "avgTaskTime": user_data["avg_time"],
            "failedLogins": user_data["failed_logins"]
        }
    )
    return response.json()
```

JavaScript usage:

```javascript
const predictRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.id,
      loginFrequency: userData.loginCount,
      taskCompletionRate: userData.completionRate,
      avgTaskTime: userData.avgTime,
      failedLogins: userData.failedLogins
    })
  });
  const result = await response.json();
  // result: { riskScore: 0.75, riskLevel: 'high', factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/detect-anomaly
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: activityData.userId,
      timestamp: activityData.timestamp,
      action: activityData.action,
      ipAddress: activityData.ip,
      location: activityData.location
    })
  });
  const result = await response.json();
  // result: { isAnomaly: true, confidence: 0.92, reason: 'Unusual login location' }
  return result;
};
```

### Burnout Detection

```javascript
// POST /api/ml/detect-burnout
const detectBurnout = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
    method: 'POST',
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json' 
    },
    body: JSON.stringify({ userId })
  });
  const result = await response.json();
  // result: { burnoutScore: 0.68, level: 'moderate', recommendations: [...] }
  return result;
};
```

### Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketData) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description
    })
  });
  const result = await response.json();
  // result: { category: 'technical', priority: 'high', suggestedAssignee: 'tech-team' }
  return result;
};
```

### Project Delay Prediction

```javascript
// POST /api/ml/predict-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.id,
      tasksTotal: projectData.totalTasks,
      tasksCompleted: projectData.completedTasks,
      teamSize: projectData.teamSize,
      deadline: projectData.deadline,
      avgCompletionTime: projectData.avgTime
    })
  });
  const result = await response.json();
  // result: { delayProbability: 0.45, estimatedDelay: '3 days', recommendations: [...] }
  return result;
};
```

## Frontend React Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      fetchUserProfile(token);
    }
  }, [token]);

  const fetchUserProfile = async (authToken) => {
    try {
      const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/me`, {
        headers: { 'Authorization': `Bearer ${authToken}` }
      });
      const data = await response.json();
      setUser(data.user);
    } catch (error) {
      console.error('Failed to fetch user:', error);
      logout();
    }
  };

  const login = async (email, password) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    
    if (data.token) {
      localStorage.setItem('token', data.token);
      setToken(data.token);
      setUser(data.user);
    }
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const KanbanBoard = () => {
  const { token, user } = useContext(AuthContext);
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${user.id}`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const data = await response.json();
    
    // Group tasks by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} onMove={(id) => moveTask(id, 'in-progress')} />
      <Column title="In Progress" tasks={tasks.inProgress} onMove={(id) => moveTask(id, 'done')} />
      <Column title="Done" tasks={tasks.done} />
    </div>
  );
};
```

### Admin Analytics Dashboard

```javascript
// src/components/AdminDashboard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const AdminDashboard = () => {
  const { token } = useContext(AuthContext);
  const [analytics, setAnalytics] = useState({});
  const [risks, setRisks] = useState([]);

  useEffect(() => {
    fetchAnalytics();
    fetchRiskUsers();
  }, []);

  const fetchAnalytics = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/admin/analytics`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const data = await response.json();
    setAnalytics(data);
  };

  const fetchRiskUsers = async () => {
    const usersResponse = await fetch(
      `${process.env.REACT_APP_API_URL}/api/users`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const users = await usersResponse.json();

    // Get risk predictions for each user
    const riskPromises = users.map(user =>
      fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/predict-risk`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          userId: user.id,
          loginFrequency: user.loginCount,
          taskCompletionRate: user.completionRate,
          avgTaskTime: user.avgTaskTime,
          failedLogins: user.failedLogins
        })
      }).then(r => r.json())
    );

    const riskResults = await Promise.all(riskPromises);
    const highRisk = riskResults
      .filter(r => r.riskLevel === 'high')
      .map((r, i) => ({ ...users[i], ...r }));
    
    setRisks(highRisk);
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      <div className="analytics-cards">
        <Card title="Total Users" value={analytics.totalUsers} />
        <Card title="Active Tasks" value={analytics.activeTasks} />
        <Card title="Open Tickets" value={analytics.openTickets} />
        <Card title="High Risk Users" value={risks.length} alert />
      </div>
      
      <div className="risk-alerts">
        <h2>High Risk Users</h2>
        {risks.map(user => (
          <RiskAlert key={user.id} user={user} />
        ))}
      </div>
    </div>
  );
};
```

## Common Patterns

### Protected Routes

```javascript
// src/components/ProtectedRoute.js
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, token } = useContext(AuthContext);

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user?.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Time Tracking Hook

```javascript
// src/hooks/useTimeTracker.js
import { useState, useEffect } from 'react';

export const useTimeTracker = (taskId, token) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const start = () => setIsRunning(true);
  
  const stop = async () => {
    setIsRunning(false);
    // Save time to backend
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ timeSpent: seconds })
    });
  };

  const reset = () => {
    setSeconds(0);
    setIsRunning(false);
  };

  return { seconds, isRunning, start, stop, reset };
};
```

## Configuration

### Backend Environment Variables

```env
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt

# Authentication
JWT_SECRET=your_jwt_secret_minimum_32_chars
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
```

### ML Service Environment Variables

```env
# Service
LOG_LEVEL=INFO
MODEL_PATH=./models

# Backend integration
BACKEND_URL=http://localhost:5000

# Model parameters
RISK_THRESHOLD=0.7
ANOMALY_THRESHOLD=0.8
BURNOUT_THRESHOLD=0.6
```

### Frontend Environment Variables

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENV=development
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### JWT Token Expiration

```javascript
// Handle token refresh
const refreshToken = async () => {
  const oldToken = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/refresh`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${oldToken}`,
      'Content-Type': 'application/json'
    }
  });
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data.token;
};
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Model Not Loading

```python
# ml-service/main.py
import os
from pathlib import Path

MODEL_PATH = Path(os.getenv('MODEL_PATH', './models'))

# Ensure model directory exists
MODEL_PATH.mkdir(exist_ok=True)

# Check if models exist, otherwise use defaults
if not (MODEL_PATH / 'risk_model.pkl').exists():
    print("Warning: Risk model not found, using default")
    # Initialize with default model
```

### Rate Limiting

```javascript
// backend/middleware/rateLimiter.js
const rateLimit = require('express-rate-limit');

const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: 'Too many requests, please try again later'
});

app.use('/api/', apiLimiter);
```
