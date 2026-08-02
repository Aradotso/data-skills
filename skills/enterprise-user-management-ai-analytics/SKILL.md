---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "Set up enterprise user management with AI analytics"
  - "Create user management system with AI features"
  - "Implement AI-powered user analytics dashboard"
  - "Build enterprise task management with AI insights"
  - "Add AI-based ticket classification system"
  - "Deploy user management with anomaly detection"
  - "Integrate AI analytics into user management"
  - "Configure enterprise user system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application combining React frontend, Node.js backend, and FastAPI ML service to manage users, tasks, and support tickets with intelligent insights. The system provides AI-powered features including risk detection, anomaly detection, burnout analysis, and predictive project insights using scikit-learn and River for online learning.

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance running

### Clone and Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Setup backend
cd backend
npm install

# Setup frontend
cd ../frontend
npm install

# Setup ML service
cd ../ml-service
pip install -r requirements.txt
```

### Environment Configuration

**Backend (.env)**
```bash
# backend/.env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
PORT=5000
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend (.env)**
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

**ML Service (.env)**
```bash
# ml-service/.env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Running the System

### Start All Services

```bash
# Terminal 1: Backend
cd backend
npm start

# Terminal 2: ML Service
cd ml-service
uvicorn main:app --reload --port 8000

# Terminal 3: Frontend
cd frontend
npm start
```

## Backend API Usage

### Authentication Endpoints

```javascript
// User registration
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: 'user' // or 'admin'
    })
  });
  return await response.json();
};

// User login
const loginUser = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management Endpoints

```javascript
// Get all users (admin only)
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return await response.json();
};

// Update user
const updateUser = async (userId, userData, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(userData)
  });
  return await response.json();
};

// Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

### Task Management Endpoints

```javascript
// Create task
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
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'in-progress', 'done'
    })
  });
  return await response.json();
};

// Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};

// Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status })
  });
  return await response.json();
};
```

### Support Ticket Endpoints

```javascript
// Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return await response.json();
};

// Get all tickets (admin)
const getAllTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

## ML Service API Usage

### AI-Powered Ticket Classification

```javascript
// Classify ticket using AI
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: "Support request"
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.87 }
  return data;
};
```

### Risk Prediction

```javascript
// Predict user risk based on behavior
const predictUserRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.userId,
      loginAttempts: userData.loginAttempts,
      failedLogins: userData.failedLogins,
      lastActiveHours: userData.lastActiveHours,
      unusualActivity: userData.unusualActivity
    })
  });
  const data = await response.json();
  // Returns: { riskScore: 0.65, riskLevel: 'medium', factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: activityData.userId,
      activityPattern: activityData.pattern,
      timestamp: activityData.timestamp,
      features: {
        loginTime: activityData.loginTime,
        accessedResources: activityData.accessedResources,
        dataVolume: activityData.dataVolume
      }
    })
  });
  const data = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.82, description: '...' }
  return data;
};
```

### Burnout Detection

```javascript
// Analyze employee burnout risk
const detectBurnout = async (workloadData) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: workloadData.userId,
      tasksCompleted: workloadData.tasksCompleted,
      averageWorkHours: workloadData.averageWorkHours,
      overtimeHours: workloadData.overtimeHours,
      missedDeadlines: workloadData.missedDeadlines,
      ticketsRaised: workloadData.ticketsRaised
    })
  });
  const data = await response.json();
  // Returns: { burnoutRisk: 'high', score: 0.78, recommendations: [...] }
  return data;
};
```

### Predictive Project Insights

```javascript
// Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/project-insights', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.projectId,
      tasksTotal: projectData.tasksTotal,
      tasksCompleted: projectData.tasksCompleted,
      daysRemaining: projectData.daysRemaining,
      teamSize: projectData.teamSize,
      complexityScore: projectData.complexityScore
    })
  });
  const data = await response.json();
  // Returns: { delayProbability: 0.45, expectedDelay: 5, suggestions: [...] }
  return data;
};
```

## Frontend React Components

### Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect } from 'react';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      // Verify token and fetch user data
      fetchUserProfile(token).then(setUser);
    }
  }, [token]);

  const login = async (email, password) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    setToken(data.token);
    setUser(data.user);
    localStorage.setItem('token', data.token);
    return data;
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
  };

  return { user, token, login, logout };
};
```

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const KanbanBoard = ({ userId }) => {
  const { token } = useAuth();
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    });
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
      method: 'PATCH',
      headers: {
        Authorization: `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} onMove={moveTask} targetStatus="todo" />
      <Column title="In Progress" tasks={tasks.inProgress} onMove={moveTask} targetStatus="in-progress" />
      <Column title="Done" tasks={tasks.done} onMove={moveTask} targetStatus="done" />
    </div>
  );
};
```

### Admin Dashboard with AI Insights

```javascript
// components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const AdminDashboard = () => {
  const { token } = useAuth();
  const [analytics, setAnalytics] = useState(null);
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    fetchAnalytics();
    fetchAIAlerts();
  }, []);

  const fetchAnalytics = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/admin/analytics`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setAnalytics(await response.json());
  };

  const fetchAIAlerts = async () => {
    // Fetch AI-generated alerts for risk, anomalies, burnout
    const response = await fetch(
      `${process.env.REACT_APP_ML_API_URL}/api/ml/alerts`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setAlerts(await response.json());
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="analytics-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p>{analytics?.totalUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p>{analytics?.activeTasks}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p>{analytics?.openTickets}</p>
        </div>
      </div>

      <div className="ai-alerts">
        <h2>AI Alerts</h2>
        {alerts.map(alert => (
          <div key={alert.id} className={`alert alert-${alert.severity}`}>
            <h4>{alert.type}</h4>
            <p>{alert.message}</p>
            <small>{alert.timestamp}</small>
          </div>
        ))}
      </div>
    </div>
  );
};
```

## Common Patterns

### Protected Routes

```javascript
// components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, token } = useAuth();

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

### Time Tracking Component

```javascript
// components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId, onSave }) => {
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

  const formatTime = (secs) => {
    const hrs = Math.floor(secs / 3600);
    const mins = Math.floor((secs % 3600) / 60);
    const s = secs % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  const handleSave = () => {
    onSave(taskId, seconds);
    setSeconds(0);
    setIsRunning(false);
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={handleSave}>Save</button>
    </div>
  );
};
```

## Configuration

### Backend Server Configuration

```javascript
// backend/config/database.js
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

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({ message: 'Access token required' });
  }

  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) {
      return res.status(403).json({ message: 'Invalid token' });
    }
    req.user = user;
    next();
  });
};

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authenticateToken, requireAdmin };
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// Check MongoDB connection
// backend/server.js
const mongoose = require('mongoose');

mongoose.connection.on('error', (err) => {
  console.error('MongoDB connection error:', err);
});

mongoose.connection.on('connected', () => {
  console.log('MongoDB connected successfully');
});

mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected');
});
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');
const express = require('express');

const app = express();

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Not Responding

```python
# ml-service/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/health")
async def health_check():
    logger.info("Health check endpoint called")
    return {"status": "healthy", "service": "ml-analytics"}
```

### Token Expiration Handling

```javascript
// frontend/utils/api.js
export const apiCall = async (endpoint, options = {}) => {
  const token = localStorage.getItem('token');
  
  const response = await fetch(`${process.env.REACT_APP_API_URL}${endpoint}`, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });

  if (response.status === 401) {
    // Token expired
    localStorage.removeItem('token');
    window.location.href = '/login';
    throw new Error('Session expired');
  }

  return await response.json();
};
```

### AI Model Loading Issues

```python
# ml-service/models/loader.py
import os
import pickle
import logging

logger = logging.getLogger(__name__)

def load_model(model_name):
    model_path = os.path.join(os.getenv('MODEL_PATH', './models'), f'{model_name}.pkl')
    
    if not os.path.exists(model_path):
        logger.warning(f"Model {model_name} not found, using default")
        return None
    
    try:
        with open(model_path, 'rb') as f:
            model = pickle.load(f)
        logger.info(f"Model {model_name} loaded successfully")
        return model
    except Exception as e:
        logger.error(f"Error loading model {model_name}: {e}")
        return None
```
