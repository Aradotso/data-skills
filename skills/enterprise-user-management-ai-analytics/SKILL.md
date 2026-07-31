---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task tracking
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management system with task tracking"
  - "implement AI-powered user analytics dashboard"
  - "build admin panel with role-based access control"
  - "add AI ticket classification and risk prediction"
  - "configure JWT authentication for user management"
  - "integrate ML service for burnout detection"
  - "develop kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack web application for managing users, tasks, and support tickets with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights.

## What It Does

This system provides:
- **User Management**: CRUD operations with role-based access control
- **Task Management**: Kanban board with time tracking and progress monitoring
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Authentication**: JWT-based secure login system
- **Real-time Insights**: Performance metrics and audit logs

## Installation

### Prerequisites
```bash
# Node.js 14+ and Python 3.8+ required
node --version
python --version
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
```

Create `backend/.env`:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
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

Create `ml-service/.env`:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
CORS_ORIGINS=http://localhost:3000,http://localhost:5000
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

Create `frontend/.env`:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
# Frontend runs at http://localhost:3000
```

## Backend API Reference

### Authentication Endpoints

**User Login**
```javascript
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

**User Registration**
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
  return await response.json();
};
```

### User Management Endpoints

**Get All Users (Admin)**
```javascript
// GET /api/users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return await response.json();
};
```

**Update User**
```javascript
// PUT /api/users/:id
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};
```

**Delete User**
```javascript
// DELETE /api/users/:id
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### Task Management Endpoints

**Create Task**
```javascript
// POST /api/tasks
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
      assignedTo: taskData.assignedTo,
      priority: taskData.priority,
      dueDate: taskData.dueDate,
      status: 'todo'
    })
  });
  return await response.json();
};
```

**Update Task Status**
```javascript
// PUT /api/tasks/:id/status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'todo', 'in-progress', 'done'
  });
  return await response.json();
};
```

**Track Time**
```javascript
// POST /api/tasks/:id/time-log
const logTime = async (taskId, timeData, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time-log`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      duration: timeData.duration, // in seconds
      date: new Date().toISOString()
    })
  });
  return await response.json();
};
```

### Support Ticket Endpoints

**Create Ticket**
```javascript
// POST /api/tickets
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
      priority: ticketData.priority
    })
  });
  return await response.json();
};
```

**Get User Tickets**
```javascript
// GET /api/tickets/my-tickets
const getMyTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets/my-tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

## ML Service API Reference

### AI Ticket Classification
```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ 
      text: ticketText 
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.89 }
  return result;
};
```

### Risk Prediction
```javascript
// POST /api/ml/predict-risk
const predictUserRisk = async (userId, userData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      failed_logins: userData.failedLogins,
      activity_hours: userData.activityHours,
      data_access: userData.dataAccess,
      permission_changes: userData.permissionChanges
    })
  });
  const result = await response.json();
  // Returns: { risk_score: 0.75, risk_level: 'high', factors: [...] }
  return result;
};
```

### Burnout Detection
```javascript
// POST /api/ml/detect-burnout
const detectBurnout = async (userId, workloadData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      tasks_count: workloadData.tasksCount,
      avg_hours_per_week: workloadData.avgHoursPerWeek,
      overdue_tasks: workloadData.overdueTasks,
      last_break_days: workloadData.lastBreakDays
    })
  });
  const result = await response.json();
  // Returns: { burnout_risk: 0.65, level: 'moderate', recommendations: [...] }
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
      timestamp: new Date().toISOString(),
      user_id: activityData.userId,
      action: activityData.action,
      ip_address: activityData.ipAddress,
      location: activityData.location
    })
  });
  const result = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.92, reason: '...' }
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
      project_id: projectData.projectId,
      tasks_completed: projectData.tasksCompleted,
      tasks_remaining: projectData.tasksRemaining,
      days_remaining: projectData.daysRemaining,
      team_size: projectData.teamSize,
      avg_completion_rate: projectData.avgCompletionRate
    })
  });
  const result = await response.json();
  // Returns: { delay_probability: 0.68, estimated_delay_days: 5 }
  return result;
};
```

## Frontend Components

### Authentication Context
```javascript
// src/contexts/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      fetchUserProfile();
    }
  }, [token]);

  const fetchUserProfile = async () => {
    try {
      const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/profile`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data);
    } catch (error) {
      console.error('Failed to fetch profile:', error);
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
      setToken(data.token);
      localStorage.setItem('token', data.token);
      setUser(data.user);
      return { success: true };
    }
    return { success: false, message: data.message };
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
// src/components/KanbanBoard.jsx
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../contexts/AuthContext';

const KanbanBoard = () => {
  const { token } = useContext(AuthContext);
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    });
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
      method: 'PUT',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  const Column = ({ title, tasks, status }) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      {tasks.map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <select value={status} onChange={(e) => moveTask(task._id, e.target.value)}>
            <option value="todo">To Do</option>
            <option value="in-progress">In Progress</option>
            <option value="done">Done</option>
          </select>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} status="todo" />
      <Column title="In Progress" tasks={tasks.inProgress} status="in-progress" />
      <Column title="Done" tasks={tasks.done} status="done" />
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard
```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../contexts/AuthContext';

const AIAnalytics = () => {
  const { token, user } = useContext(AuthContext);
  const [analytics, setAnalytics] = useState({
    burnoutRisk: null,
    riskScore: null,
    projectDelays: []
  });

  useEffect(() => {
    if (user) {
      fetchAnalytics();
    }
  }, [user]);

  const fetchAnalytics = async () => {
    // Fetch burnout detection
    const burnoutResponse = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/detect-burnout`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        user_id: user._id,
        tasks_count: user.taskStats?.total || 0,
        avg_hours_per_week: user.workStats?.avgHours || 40,
        overdue_tasks: user.taskStats?.overdue || 0,
        last_break_days: user.workStats?.daysSinceBreak || 0
      })
    });
    const burnoutData = await burnoutResponse.json();

    // Fetch risk prediction
    const riskResponse = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/predict-risk`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        user_id: user._id,
        failed_logins: user.securityStats?.failedLogins || 0,
        activity_hours: user.activityStats?.totalHours || 0,
        data_access: user.securityStats?.dataAccess || 0,
        permission_changes: user.securityStats?.permissionChanges || 0
      })
    });
    const riskData = await riskResponse.json();

    setAnalytics({
      burnoutRisk: burnoutData,
      riskScore: riskData,
      projectDelays: []
    });
  };

  return (
    <div className="ai-analytics">
      <h2>AI Analytics Dashboard</h2>
      
      {analytics.burnoutRisk && (
        <div className="burnout-widget">
          <h3>Burnout Risk: {analytics.burnoutRisk.level}</h3>
          <p>Score: {(analytics.burnoutRisk.burnout_risk * 100).toFixed(0)}%</p>
          <ul>
            {analytics.burnoutRisk.recommendations?.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      )}

      {analytics.riskScore && (
        <div className="risk-widget">
          <h3>Security Risk: {analytics.riskScore.risk_level}</h3>
          <p>Score: {(analytics.riskScore.risk_score * 100).toFixed(0)}%</p>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Common Patterns

### Protected Route Wrapper
```javascript
// src/components/ProtectedRoute.jsx
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../contexts/AuthContext';

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
import { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../contexts/AuthContext';

export const useTimeTracker = (taskId) => {
  const { token } = useContext(AuthContext);
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
  const pause = () => setIsRunning(false);
  const reset = () => {
    setSeconds(0);
    setIsRunning(false);
  };

  const saveLog = async () => {
    if (seconds > 0) {
      await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time-log`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ duration: seconds })
      });
      reset();
    }
  };

  return { seconds, isRunning, start, pause, reset, saveLog };
};
```

## Configuration

### Backend Environment Variables
```env
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management

# JWT
JWT_SECRET=your_secure_random_string
JWT_EXPIRE=7d

# External Services
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
```

### ML Service Configuration
```python
# ml-service/config.py
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    MODEL_PATH: str = os.getenv('MODEL_PATH', './models')
    LOG_LEVEL: str = os.getenv('LOG_LEVEL', 'INFO')
    CORS_ORIGINS: list = os.getenv('CORS_ORIGINS', 'http://localhost:3000').split(',')
    
    RISK_THRESHOLD_LOW: float = 0.3
    RISK_THRESHOLD_MEDIUM: float = 0.6
    RISK_THRESHOLD_HIGH: float = 0.8
    
    BURNOUT_THRESHOLD_LOW: float = 0.4
    BURNOUT_THRESHOLD_MEDIUM: float = 0.7
    BURNOUT_THRESHOLD_HIGH: float = 0.85

settings = Settings()
```

## Troubleshooting

### Backend Won't Start
```bash
# Check if MongoDB is running
sudo systemctl status mongod

# Start MongoDB if not running
sudo systemctl start mongod

# Check port availability
lsof -i :5000

# Clear node_modules and reinstall
rm -rf node_modules package-lock.json
npm install
```

### JWT Token Errors
```javascript
// Ensure token is being sent correctly
const makeAuthRequest = async (url, options = {}) => {
  const token = localStorage.getItem('token');
  
  if (!token) {
    throw new Error('No authentication token found');
  }

  const response = await fetch(url, {
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
  }

  return response;
};
```

### ML Service Connection Issues
```python
# ml-service/main.py - Add CORS middleware
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### MongoDB Connection Errors
```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Frontend Build Errors
```bash
# Clear cache
rm -rf node_modules/.cache

# Update dependencies
npm update

# Check for conflicting packages
npm ls

# Build with verbose logging
npm run build --verbose
```

### AI Model Not Loading
```python
# ml-service/main.py - Check model initialization
import os
import pickle
from pathlib import Path

def load_models():
    model_path = Path(os.getenv('MODEL_PATH', './models'))
    
    if not model_path.exists():
        model_path.mkdir(parents=True, exist_ok=True)
        print(f"Created model directory at {model_path}")
        return None
    
    try:
        with open(model_path / 'risk_model.pkl', 'rb') as f:
            risk_model = pickle.load(f)
        return risk_model
    except FileNotFoundError:
        print("Model file not found. Training new model...")
        # Initialize and train a new model
        return train_new_model()
```
