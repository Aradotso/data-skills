---
name: enterprise-user-management-ai-analytics
description: Enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management system with machine learning"
  - "implement AI-powered task tracking and user administration"
  - "build enterprise dashboard with anomaly detection"
  - "add AI analytics to user management platform"
  - "integrate ML-based risk prediction for users"
  - "develop full-stack user management with FastAPI ML service"
  - "create kanban board with burnout detection"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user administration, task management, and support ticket handling with AI-powered insights. The system uses machine learning for risk detection, anomaly detection, burnout analysis, and predictive project insights to help organizations automate workflows and improve decision-making.

**Architecture:**
- **Frontend**: React.js dashboard for admins and users
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI service using scikit-learn and River for online learning
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

Ensure you have installed:
- Node.js (v14+)
- Python (3.8+)
- MongoDB (running locally or cloud instance)

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

Create `.env` file in `backend/`:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in `ml-service/`:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in `frontend/`:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Frontend runs at http://localhost:3000
```

## Key API Endpoints

### Authentication APIs

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
      role: userData.role // 'admin' or 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
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
  // Returns: { token: 'jwt_token', user: {...} }
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management APIs

```javascript
// GET /api/users (Admin only)
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PUT /api/users/:id (Admin only)
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id (Admin only)
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management APIs

```javascript
// POST /api/tasks
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
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
  return response.json();
};

// GET /api/tasks/user/:userId
const getUserTasks = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};
```

### Support Ticket APIs

```javascript
// POST /api/tickets
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
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
  return response.json();
};

// GET /api/tickets
const getAllTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## AI/ML Service Integration

### Risk Prediction

```javascript
// POST /predict/risk
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/predict/risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      features: {
        login_frequency: 45,
        failed_logins: 3,
        task_completion_rate: 0.75,
        avg_response_time: 120,
        ticket_count: 5
      }
    })
  });
  const data = await response.json();
  // Returns: { risk_score: 0.23, risk_level: 'low', factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// POST /detect/anomaly
const detectAnomaly = async (userActivity) => {
  const response = await fetch('http://localhost:8000/detect/anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userActivity.userId,
      activity_data: {
        login_time: userActivity.loginTime,
        ip_address: userActivity.ipAddress,
        location: userActivity.location,
        device: userActivity.device,
        actions: userActivity.actions
      }
    })
  });
  const data = await response.json();
  // Returns: { is_anomaly: false, anomaly_score: 0.12, details: {...} }
  return data;
};
```

### Burnout Detection

```javascript
// POST /predict/burnout
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/predict/burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      metrics: {
        tasks_assigned: 25,
        tasks_completed: 18,
        avg_task_duration: 180, // minutes
        overtime_hours: 15,
        weekend_work_frequency: 3,
        break_frequency: 2
      }
    })
  });
  const data = await response.json();
  // Returns: { burnout_risk: 'medium', score: 0.65, recommendations: [...] }
  return data;
};
```

### Ticket Classification

```javascript
// POST /classify/ticket
const classifyTicket = async (ticketContent) => {
  const response = await fetch('http://localhost:8000/classify/ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      subject: ticketContent.subject,
      description: ticketContent.description
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', suggested_assignee: 'team-a' }
  return data;
};
```

### Predictive Project Insights

```javascript
// POST /predict/project-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/predict/project-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.projectId,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      days_remaining: projectData.daysRemaining,
      team_size: projectData.teamSize,
      avg_velocity: projectData.avgVelocity
    })
  });
  const data = await response.json();
  // Returns: { delay_probability: 0.45, estimated_delay_days: 3, risk_factors: [...] }
  return data;
};
```

## Common Usage Patterns

### Protected Route Component (React)

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/dashboard" element={
          <ProtectedRoute>
            <UserDashboard />
          </ProtectedRoute>
        } />
        <Route path="/admin" element={
          <ProtectedRoute requiredRole="admin">
            <AdminDashboard />
          </ProtectedRoute>
        } />
      </Routes>
    </BrowserRouter>
  );
}
```

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    // Organize tasks by status
    const organized = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(organized);
  };

  const moveTask = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks(); // Refresh board
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} onMove={moveTask} targetStatus="in-progress" />
      <Column title="In Progress" tasks={tasks.inProgress} onMove={moveTask} targetStatus="done" />
      <Column title="Done" tasks={tasks.done} />
    </div>
  );
};
```

### Time Tracking Hook

```javascript
import { useState, useEffect } from 'react';

const useTimeTracker = (taskId) => {
  const [isTracking, setIsTracking] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);

  useEffect(() => {
    let interval;
    if (isTracking) {
      interval = setInterval(() => {
        setElapsedTime(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isTracking]);

  const startTracking = () => {
    setIsTracking(true);
  };

  const stopTracking = async () => {
    setIsTracking(false);
    
    // Log time to backend
    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/tasks/${taskId}/log-time`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ duration: elapsedTime })
    });
  };

  const resetTracking = () => {
    setElapsedTime(0);
    setIsTracking(false);
  };

  return { isTracking, elapsedTime, startTracking, stopTracking, resetTracking };
};

export default useTimeTracker;
```

### AI-Powered User Analytics Dashboard

```javascript
import React, { useState, useEffect } from 'react';

const UserAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch risk prediction
      const riskRes = await fetch('http://localhost:8000/predict/risk', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: userId })
      });
      const riskData = await riskRes.json();

      // Fetch burnout detection
      const burnoutRes = await fetch('http://localhost:8000/predict/burnout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: userId })
      });
      const burnoutData = await burnoutRes.json();

      setAnalytics({
        riskScore: riskData,
        burnoutRisk: burnoutData,
        anomalies: []
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="analytics-dashboard">
      <div className="risk-indicator">
        <h3>Risk Level: {analytics.riskScore?.risk_level}</h3>
        <div className="score">{analytics.riskScore?.risk_score}</div>
      </div>
      <div className="burnout-indicator">
        <h3>Burnout Risk: {analytics.burnoutRisk?.burnout_risk}</h3>
        <div className="score">{analytics.burnoutRisk?.score}</div>
        <ul>
          {analytics.burnoutRisk?.recommendations?.map((rec, idx) => (
            <li key={idx}>{rec}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};
```

## Configuration

### Backend Configuration (`backend/config.js`)

```javascript
module.exports = {
  port: process.env.PORT || 5000,
  mongoURI: process.env.MONGODB_URI,
  jwtSecret: process.env.JWT_SECRET,
  jwtExpire: process.env.JWT_EXPIRE || '7d',
  mlServiceURL: process.env.ML_SERVICE_URL || 'http://localhost:8000',
  
  // Rate limiting
  rateLimitWindowMs: 15 * 60 * 1000, // 15 minutes
  rateLimitMax: 100,
  
  // File upload
  maxFileSize: 5 * 1024 * 1024, // 5MB
  
  // Email (if configured)
  emailService: process.env.EMAIL_SERVICE,
  emailUser: process.env.EMAIL_USER,
  emailPassword: process.env.EMAIL_PASSWORD
};
```

### ML Service Configuration

Create `ml-service/config.py`:

```python
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    MONGODB_URI = os.getenv('MONGODB_URI', 'mongodb://localhost:27017/enterprise-user-mgmt')
    MODEL_PATH = os.getenv('MODEL_PATH', './models')
    LOG_LEVEL = os.getenv('LOG_LEVEL', 'INFO')
    
    # ML Model Parameters
    RISK_THRESHOLD = 0.7
    ANOMALY_THRESHOLD = 0.5
    BURNOUT_THRESHOLD = 0.6
    
    # Online learning
    RETRAIN_INTERVAL = 86400  # 24 hours in seconds
    MIN_SAMPLES_RETRAIN = 100

config = Config()
```

## Troubleshooting

### Issue: JWT Token Expired

```javascript
// Add token refresh logic
const refreshToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  const response = await fetch('http://localhost:5000/api/auth/refresh', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken })
  });
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data.token;
};

// Axios interceptor for auto-refresh
import axios from 'axios';

axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      try {
        const newToken = await refreshToken();
        error.config.headers.Authorization = `Bearer ${newToken}`;
        return axios(error.config);
      } catch (refreshError) {
        // Redirect to login
        window.location.href = '/login';
      }
    }
    return Promise.reject(error);
  }
);
```

### Issue: ML Service Connection Failed

```javascript
// Add fallback when ML service is unavailable
const getPredictionWithFallback = async (endpoint, data) => {
  try {
    const response = await fetch(`http://localhost:8000${endpoint}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
      timeout: 5000
    });
    
    if (!response.ok) throw new Error('ML service error');
    return await response.json();
  } catch (error) {
    console.warn('ML service unavailable, using fallback:', error);
    // Return default/cached predictions
    return { 
      available: false, 
      message: 'AI analytics temporarily unavailable' 
    };
  }
};
```

### Issue: MongoDB Connection Error

```javascript
// backend/db.js - Add retry logic
const mongoose = require('mongoose');

const connectDB = async (retries = 5) => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB Connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    
    if (retries > 0) {
      console.log(`Retrying connection... (${retries} attempts left)`);
      setTimeout(() => connectDB(retries - 1), 5000);
    } else {
      process.exit(1);
    }
  }
};

module.exports = connectDB;
```

### Issue: CORS Errors

```javascript
// backend/server.js - Configure CORS properly
const express = require('express');
const cors = require('cors');

const app = express();

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Issue: Model Performance Degradation

```python
# ml-service/main.py - Add model monitoring
from datetime import datetime
import logging

class ModelMonitor:
    def __init__(self):
        self.predictions = []
        self.performance_log = []
    
    def log_prediction(self, input_data, prediction, actual=None):
        entry = {
            'timestamp': datetime.now(),
            'input': input_data,
            'prediction': prediction,
            'actual': actual
        }
        self.predictions.append(entry)
        
        # Check if model needs retraining
        if len(self.predictions) >= 100:
            self.evaluate_performance()
    
    def evaluate_performance(self):
        # Calculate accuracy, drift, etc.
        if self.should_retrain():
            logging.warning('Model performance degraded, triggering retrain')
            # Trigger retraining pipeline

monitor = ModelMonitor()
```

This skill provides comprehensive coverage for developers to use the Enterprise User Management System with AI Analytics, including API integration, ML service usage, and common implementation patterns.
