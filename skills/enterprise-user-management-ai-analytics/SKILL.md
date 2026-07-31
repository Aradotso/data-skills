---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "build user management with AI analytics"
  - "implement AI-powered task management"
  - "create admin dashboard with analytics"
  - "add AI ticket classification system"
  - "integrate burnout detection features"
  - "deploy user management with JWT auth"
  - "configure ML service for user insights"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack application combining user/task management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights. Built with React, Node.js, FastAPI, and MongoDB.

## What This Project Does

This system provides:
- **User Management**: Role-based access control, JWT authentication, user CRUD operations
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support System**: Ticket creation, tracking, and AI-based classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, alerts

## Installation

### Prerequisites
- Node.js 14+
- Python 3.8+
- MongoDB instance
- npm or yarn

### Clone and Install

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install
```

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
# or for development
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MONGODB_URI=your_mongodb_connection_string
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Backend API Usage

### Authentication Endpoints

```javascript
// Register new user (Admin only)
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${adminToken}`
    },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role, // 'admin' or 'user'
      department: userData.department
    })
  });
  return await response.json();
};

// Login
const login = async (email, password) => {
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

### User Management

```javascript
// Get all users (Admin)
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};

// Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData, token) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
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

// Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status })
  });
  return await response.json();
};

// Track time on task
const trackTime = async (taskId, timeSpent, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ timeSpent }) // in minutes
  });
  return await response.json();
};
```

### Ticket Management

```javascript
// Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
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

// Get user tickets
const getUserTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets/my-tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

## ML Service API Usage

### AI Ticket Classification

```javascript
// Classify ticket using AI
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: "Subject line"
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.87 }
  return data;
};
```

### Risk Prediction

```javascript
// Predict user risk score
const predictRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ userId })
  });
  const data = await response.json();
  // Returns: { riskScore: 0.72, factors: ['high_workload', 'missed_deadlines'], recommendation: 'Reduce workload' }
  return data;
};
```

### Anomaly Detection

```javascript
// Detect anomalous behavior
const detectAnomaly = async (userActivity) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userActivity.userId,
      loginTime: userActivity.loginTime,
      location: userActivity.location,
      activityPattern: userActivity.pattern
    })
  });
  const data = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.85, reason: 'Unusual login time' }
  return data;
};
```

### Burnout Analysis

```javascript
// Analyze user burnout risk
const analyzeBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-analysis', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ userId })
  });
  const data = await response.json();
  // Returns: { burnoutScore: 0.68, indicators: ['overtime_hours', 'task_overload'], suggestion: 'Schedule time off' }
  return data;
};
```

### Predictive Project Insights

```javascript
// Predict project delay
const predictProjectDelay = async (projectId) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ projectId })
  });
  const data = await response.json();
  // Returns: { delayProbability: 0.62, estimatedDelay: '5 days', factors: ['resource_shortage', 'complexity'] }
  return data;
};
```

## React Frontend Patterns

### Authentication Context

```javascript
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      // Verify token and get user data
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
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
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
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from './AuthContext';

const KanbanBoard = () => {
  const { token } = useContext(AuthContext);
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/my-tasks`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
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
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
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
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from './AuthContext';

const AdminDashboard = () => {
  const { token } = useContext(AuthContext);
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);

  useEffect(() => {
    fetchAnalytics();
    fetchRiskUsers();
  }, []);

  const fetchAnalytics = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/admin/analytics`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    setAnalytics(await response.json());
  };

  const fetchRiskUsers = async () => {
    const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/high-risk-users`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    setRiskUsers(await response.json());
  };

  return (
    <div className="admin-dashboard">
      <h1>Organization Analytics</h1>
      {analytics && (
        <div className="stats">
          <div>Total Users: {analytics.totalUsers}</div>
          <div>Active Tasks: {analytics.activeTasks}</div>
          <div>Open Tickets: {analytics.openTickets}</div>
          <div>Completion Rate: {analytics.completionRate}%</div>
        </div>
      )}
      
      <h2>High Risk Users</h2>
      <ul>
        {riskUsers.map(user => (
          <li key={user.userId}>
            {user.name} - Risk Score: {user.riskScore.toFixed(2)}
            <span>Factors: {user.factors.join(', ')}</span>
          </li>
        ))}
      </ul>
    </div>
  );
};
```

## Configuration

### Backend Environment Variables

```env
# Server
PORT=5000
NODE_ENV=production

# Database
MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/dbname

# JWT
JWT_SECRET=your_secure_random_string_here
JWT_EXPIRE=7d

# External Services
ML_SERVICE_URL=http://localhost:8000

# Email (if configured)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email
SMTP_PASS=your_app_password
```

### ML Service Environment Variables

```env
# Service
LOG_LEVEL=INFO
MODEL_PATH=./models

# Database
MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/dbname

# ML Settings
RISK_THRESHOLD=0.7
ANOMALY_THRESHOLD=0.8
BURNOUT_THRESHOLD=0.65
```

### Frontend Environment Variables

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENVIRONMENT=development
```

## Common Patterns

### Protected Routes

```javascript
import { Navigate } from 'react-router-dom';
import { useContext } from 'react';
import { AuthContext } from './AuthContext';

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

// Usage in App.js
<Route path="/admin" element={
  <ProtectedRoute adminOnly={true}>
    <AdminDashboard />
  </ProtectedRoute>
} />
```

### Real-time Notifications

```javascript
import { useEffect, useState, useContext } from 'react';
import { AuthContext } from './AuthContext';

const useNotifications = () => {
  const { token, user } = useContext(AuthContext);
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    if (!token) return;

    // Poll for notifications every 30 seconds
    const interval = setInterval(async () => {
      const response = await fetch(`${process.env.REACT_APP_API_URL}/api/notifications`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setNotifications(data);
    }, 30000);

    return () => clearInterval(interval);
  }, [token]);

  return notifications;
};
```

### Time Tracking Component

```javascript
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

  const handleSave = async () => {
    const minutes = Math.floor(seconds / 60);
    await onSave(taskId, minutes);
    setSeconds(0);
    setIsRunning(false);
  };

  const formatTime = (secs) => {
    const hrs = Math.floor(secs / 3600);
    const mins = Math.floor((secs % 3600) / 60);
    const secs_left = secs % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs_left.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="display">{formatTime(seconds)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={handleSave} disabled={seconds === 0}>
        Save Time
      </button>
    </div>
  );
};
```

## Troubleshooting

### JWT Token Expiration

If getting "Token expired" errors:

```javascript
// Add token refresh logic
const refreshToken = async () => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/refresh`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${localStorage.getItem('token')}`
    }
  });
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data.token;
};

// Use axios interceptor or fetch wrapper
const fetchWithAuth = async (url, options = {}) => {
  let token = localStorage.getItem('token');
  
  const response = await fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`
    }
  });

  if (response.status === 401) {
    token = await refreshToken();
    return fetch(url, {
      ...options,
      headers: {
        ...options.headers,
        'Authorization': `Bearer ${token}`
      }
    });
  }

  return response;
};
```

### MongoDB Connection Issues

Check connection string format:

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};
```

### ML Service Not Responding

Ensure FastAPI is properly configured:

```python
# ml-service/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import uvicorn

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### CORS Issues

Backend CORS configuration:

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');

const app = express();

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Model Loading Errors

Verify model files exist and paths are correct:

```python
# ml-service/utils/model_loader.py
import os
import pickle
from pathlib import Path

def load_model(model_name):
    model_path = Path(os.getenv('MODEL_PATH', './models')) / f"{model_name}.pkl"
    
    if not model_path.exists():
        raise FileNotFoundError(f"Model not found: {model_path}")
    
    with open(model_path, 'rb') as f:
        return pickle.load(f)
```
