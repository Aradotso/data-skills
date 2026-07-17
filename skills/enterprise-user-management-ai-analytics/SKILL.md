---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management for enterprise organizations
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement task tracking with burnout detection"
  - "create admin dashboard with user analytics"
  - "build support ticket system with AI routing"
  - "deploy user management with risk prediction"
  - "configure AI-powered enterprise management"
  - "implement kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack application for managing enterprise users, tasks, and support tickets with integrated AI capabilities including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards, time tracking, assignment workflows
- **Support System**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Tools**: Analytics dashboards, audit logs, user monitoring

The system consists of three main components:
1. **Frontend** (React.js) - User and admin interfaces
2. **Backend** (Node.js/Express) - REST API and business logic
3. **ML Service** (FastAPI) - AI/ML models and predictions

## Installation

### Prerequisites
- Node.js 14+ and npm
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
MONGODB_URI=mongodb://localhost:27017/user_management
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
```

Start backend:
```bash
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:
```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
# Runs at http://localhost:3000
```

## Backend API Reference

### Authentication Endpoints

**Register User**
```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch(`${API_URL}/api/auth/register`, {
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
```

**Login**
```javascript
// POST /api/auth/login
const loginUser = async (email, password) => {
  const response = await fetch(`${API_URL}/api/auth/login`, {
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

### User Management Endpoints

**Get All Users (Admin)**
```javascript
// GET /api/users
const getAllUsers = async (token) => {
  const response = await fetch(`${API_URL}/api/users`, {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};
```

**Update User**
```javascript
// PUT /api/users/:id
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`${API_URL}/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

**Delete User**
```javascript
// DELETE /api/users/:id
const deleteUser = async (userId, token) => {
  const response = await fetch(`${API_URL}/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management Endpoints

**Create Task**
```javascript
// POST /api/tasks
const createTask = async (taskData, token) => {
  const response = await fetch(`${API_URL}/api/tasks`, {
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
```

**Get User Tasks**
```javascript
// GET /api/tasks/user/:userId
const getUserTasks = async (userId, token) => {
  const response = await fetch(`${API_URL}/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

**Update Task Status**
```javascript
// PATCH /api/tasks/:id/status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`${API_URL}/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'todo', 'in-progress', 'done'
  });
  return response.json();
};
```

**Track Time**
```javascript
// POST /api/tasks/:id/time
const trackTaskTime = async (taskId, timeData, token) => {
  const response = await fetch(`${API_URL}/api/tasks/${taskId}/time`, {
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
  return response.json();
};
```

### Support Ticket Endpoints

**Create Ticket**
```javascript
// POST /api/tickets
const createTicket = async (ticketData, token) => {
  const response = await fetch(`${API_URL}/api/tickets`, {
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
  return response.json();
};
```

**Get Tickets**
```javascript
// GET /api/tickets
const getTickets = async (filters, token) => {
  const params = new URLSearchParams(filters);
  const response = await fetch(`${API_URL}/api/tickets?${params}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## ML Service API Reference

### Risk Prediction

**Predict User Risk**
```javascript
// POST /api/ml/risk-prediction
const predictRisk = async (userData, token) => {
  const response = await fetch(`${ML_SERVICE_URL}/api/ml/risk-prediction`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId: userData.userId,
      taskCount: userData.taskCount,
      completionRate: userData.completionRate,
      avgResponseTime: userData.avgResponseTime,
      loginFrequency: userData.loginFrequency
    })
  });
  return response.json();
  // Returns: { risk_score: 0.75, risk_level: "high", factors: [...] }
};
```

### Anomaly Detection

**Detect Anomalies**
```javascript
// POST /api/ml/anomaly-detection
const detectAnomalies = async (activityData, token) => {
  const response = await fetch(`${ML_SERVICE_URL}/api/ml/anomaly-detection`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId: activityData.userId,
      loginTime: activityData.loginTime,
      location: activityData.location,
      activityPattern: activityData.activityPattern
    })
  });
  return response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.85, details: {...} }
};
```

### Burnout Detection

**Analyze Burnout Risk**
```javascript
// POST /api/ml/burnout-detection
const detectBurnout = async (workloadData, token) => {
  const response = await fetch(`${ML_SERVICE_URL}/api/ml/burnout-detection`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId: workloadData.userId,
      hoursWorked: workloadData.hoursWorked,
      tasksCompleted: workloadData.tasksCompleted,
      overtimeHours: workloadData.overtimeHours,
      weekendWork: workloadData.weekendWork,
      taskDeadlinesMissed: workloadData.taskDeadlinesMissed
    })
  });
  return response.json();
  // Returns: { burnout_risk: "high", score: 0.82, recommendations: [...] }
};
```

### Ticket Classification

**Classify Support Ticket**
```javascript
// POST /api/ml/ticket-classification
const classifyTicket = async (ticketText, token) => {
  const response = await fetch(`${ML_SERVICE_URL}/api/ml/ticket-classification`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketText.subject,
      description: ticketText.description
    })
  });
  return response.json();
  // Returns: { category: "technical", priority: "high", suggested_assignee: "..." }
};
```

### Project Insights

**Predict Project Delays**
```javascript
// POST /api/ml/project-insights
const predictProjectDelay = async (projectData, token) => {
  const response = await fetch(`${ML_SERVICE_URL}/api/ml/project-insights`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      projectId: projectData.projectId,
      totalTasks: projectData.totalTasks,
      completedTasks: projectData.completedTasks,
      daysRemaining: projectData.daysRemaining,
      teamSize: projectData.teamSize,
      avgCompletionRate: projectData.avgCompletionRate
    })
  });
  return response.json();
  // Returns: { delay_probability: 0.65, estimated_delay_days: 5, risk_factors: [...] }
};
```

## Common Usage Patterns

### Complete User Authentication Flow

```javascript
import React, { useState, useEffect } from 'react';

const API_URL = process.env.REACT_APP_API_URL;

const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      validateToken();
    }
  }, [token]);

  const validateToken = async () => {
    try {
      const response = await fetch(`${API_URL}/api/auth/validate`, {
        headers: {
          'Authorization': `Bearer ${token}`
        }
      });
      if (response.ok) {
        const data = await response.json();
        setUser(data.user);
      } else {
        logout();
      }
    } catch (error) {
      console.error('Token validation failed:', error);
      logout();
    }
  };

  const login = async (email, password) => {
    const response = await fetch(`${API_URL}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    
    if (response.ok) {
      setToken(data.token);
      setUser(data.user);
      localStorage.setItem('token', data.token);
      return { success: true };
    }
    return { success: false, error: data.message };
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

### Kanban Board Implementation

```javascript
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId, token }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
      {
        headers: { 'Authorization': `Bearer ${token}` }
      }
    );
    const data = await response.json();
    
    // Organize tasks by status
    const organized = {
      todo: data.filter(t => t.status === 'todo'),
      'in-progress': data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(organized);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
      {
        method: 'PATCH',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ status: newStatus })
      }
    );
    fetchTasks(); // Refresh board
  };

  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <TaskCard
              key={task._id}
              task={task}
              onStatusChange={(newStatus) => 
                updateTaskStatus(task._id, newStatus)
              }
            />
          ))}
        </div>
      ))}
    </div>
  );
};
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId, token }) => {
  const [isTracking, setIsTracking] = useState(false);
  const [seconds, setSeconds] = useState(0);
  const [startTime, setStartTime] = useState(null);

  useEffect(() => {
    let interval = null;
    if (isTracking) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isTracking]);

  const startTracking = () => {
    setIsTracking(true);
    setStartTime(Date.now());
  };

  const stopTracking = async () => {
    setIsTracking(false);
    
    // Save time to backend
    await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          duration: seconds,
          date: new Date().toISOString()
        })
      }
    );
    
    setSeconds(0);
  };

  const formatTime = (secs) => {
    const h = Math.floor(secs / 3600);
    const m = Math.floor((secs % 3600) / 60);
    const s = secs % 60;
    return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={isTracking ? stopTracking : startTracking}>
        {isTracking ? 'Stop' : 'Start'}
      </button>
    </div>
  );
};
```

### Admin Analytics Dashboard

```javascript
import React, { useState, useEffect } from 'react';

const AdminDashboard = ({ token }) => {
  const [analytics, setAnalytics] = useState(null);
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    fetchAnalytics();
    fetchAlerts();
  }, []);

  const fetchAnalytics = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/admin/analytics`,
      {
        headers: { 'Authorization': `Bearer ${token}` }
      }
    );
    const data = await response.json();
    setAnalytics(data);
  };

  const fetchAlerts = async () => {
    // Get high-risk users
    const usersResponse = await fetch(
      `${process.env.REACT_APP_API_URL}/api/users`,
      {
        headers: { 'Authorization': `Bearer ${token}` }
      }
    );
    const users = await usersResponse.json();

    // Check each user for risks
    const alertsPromises = users.map(async (user) => {
      const riskResponse = await fetch(
        `${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/risk-prediction`,
        {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            userId: user._id,
            taskCount: user.tasks?.length || 0,
            completionRate: user.completionRate || 0,
            avgResponseTime: user.avgResponseTime || 0,
            loginFrequency: user.loginFrequency || 0
          })
        }
      );
      const risk = await riskResponse.json();
      
      if (risk.risk_level === 'high') {
        return {
          type: 'risk',
          user: user.name,
          message: `High risk detected for ${user.name}`,
          score: risk.risk_score
        };
      }
      return null;
    });

    const results = await Promise.all(alertsPromises);
    setAlerts(results.filter(a => a !== null));
  };

  if (!analytics) return <div>Loading...</div>;

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <StatCard title="Total Users" value={analytics.totalUsers} />
        <StatCard title="Active Tasks" value={analytics.activeTasks} />
        <StatCard title="Open Tickets" value={analytics.openTickets} />
        <StatCard title="Completion Rate" value={`${analytics.completionRate}%`} />
      </div>

      <div className="alerts-section">
        <h2>Alerts</h2>
        {alerts.map((alert, idx) => (
          <AlertCard key={idx} alert={alert} />
        ))}
      </div>

      <div className="user-list">
        <h2>User Management</h2>
        <UserTable token={token} />
      </div>
    </div>
  );
};
```

### AI-Powered Ticket Routing

```javascript
const createAndRouteTicket = async (ticketData, token) => {
  // First, classify the ticket using AI
  const classificationResponse = await fetch(
    `${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/ticket-classification`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        subject: ticketData.subject,
        description: ticketData.description
      })
    }
  );
  const classification = await classificationResponse.json();

  // Create ticket with AI-suggested classification
  const ticketResponse = await fetch(
    `${process.env.REACT_APP_API_URL}/api/tickets`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        ...ticketData,
        category: classification.category,
        priority: classification.priority,
        assignedTo: classification.suggested_assignee
      })
    }
  );

  return ticketResponse.json();
};
```

### Burnout Monitoring Hook

```javascript
import { useState, useEffect } from 'react';

const useBurnoutMonitoring = (userId, token) => {
  const [burnoutStatus, setBurnoutStatus] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    checkBurnout();
    // Check every hour
    const interval = setInterval(checkBurnout, 3600000);
    return () => clearInterval(interval);
  }, [userId]);

  const checkBurnout = async () => {
    try {
      // Get user workload data
      const workloadResponse = await fetch(
        `${process.env.REACT_APP_API_URL}/api/users/${userId}/workload`,
        {
          headers: { 'Authorization': `Bearer ${token}` }
        }
      );
      const workload = await workloadResponse.json();

      // Analyze burnout risk
      const burnoutResponse = await fetch(
        `${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/burnout-detection`,
        {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            userId,
            hoursWorked: workload.hoursWorked,
            tasksCompleted: workload.tasksCompleted,
            overtimeHours: workload.overtimeHours,
            weekendWork: workload.weekendWork,
            taskDeadlinesMissed: workload.taskDeadlinesMissed
          })
        }
      );
      const burnout = await burnoutResponse.json();
      
      setBurnoutStatus(burnout);
      setLoading(false);
    } catch (error) {
      console.error('Burnout check failed:', error);
      setLoading(false);
    }
  };

  return { burnoutStatus, loading, refresh: checkBurnout };
};

// Usage
const UserProfile = ({ userId, token }) => {
  const { burnoutStatus, loading } = useBurnoutMonitoring(userId, token);

  if (loading) return <div>Checking wellness...</div>;

  return (
    <div className="user-profile">
      {burnoutStatus && burnoutStatus.burnout_risk === 'high' && (
        <div className="burnout-warning">
          <h3>⚠️ High Burnout Risk Detected</h3>
          <p>Score: {(burnoutStatus.score * 100).toFixed(0)}%</p>
          <ul>
            {burnoutStatus.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};
```

## Configuration

### Backend Environment Variables

```env
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/user_management
MONGODB_TEST_URI=mongodb://localhost:27017/user_management_test

# Authentication
JWT_SECRET=your_secure_jwt_secret_key_here
JWT_EXPIRE=7d
REFRESH_TOKEN_SECRET=your_refresh_token_secret
REFRESH_TOKEN_EXPIRE=30d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

# Logging
LOG_LEVEL=info
```

### ML Service Configuration

```env
# Model Settings
MODEL_PATH=./models
MODEL_RETRAIN_INTERVAL=86400  # seconds (24 hours)

# API Settings
API_RATE_LIMIT=100  # requests per minute
CORS_ORIGINS=http://localhost:3000,http://localhost:5000

# ML Parameters
RISK_THRESHOLD=0.7
ANOMALY_THRESHOLD=0.8
BURNOUT_THRESHOLD=0.75

# Logging
LOG_LEVEL=INFO
LOG_FILE=./logs/ml_service.log
```

### Frontend Environment Variables

```env
# API Endpoints
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000

# Features
REACT_APP_ENABLE_AI_FEATURES=true
REACT_APP_ENABLE_NOTIFICATIONS=true

# Build
GENERATE_SOURCEMAP=false
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    // Retry connection after 5 seconds
    setTimeout(connectDB, 5000);
  }
};

module.exports = connectDB;
```

### JWT Token Expiration Handling

```javascript
// frontend/utils/api.js
const fetchWithAuth = async (url, options = {}) => {
  const token = localStorage.getItem('token');
  
  const response = await fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`
    }
  });

  if (response.status === 401) {
    // Token expired, try to refresh
    const refreshToken = localStorage.getItem('refreshToken');
    if (refreshToken) {
      const refreshResponse = await fetch(
        `${process.env.REACT_APP_API_URL}/auth/refresh`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ refreshToken })
        }
      );
      
      if (refreshResponse.ok) {
        const { token: newToken } = await refreshResponse.json();
        localStorage.setItem('token', newToken);
        // Retry original request
        return fetchWithAuth(url, options);
      }
    }
    
    // Refresh failed, logout
    localStorage.clear();
    window.location.href = '/login';
    throw new Error('Session expired');
  }

  return response;
};
```

### ML Service Model Loading

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
import joblib
import os
from pathlib import Path

app = FastAPI()

models = {}

@app.on_event("startup")
async def load_models():
    """Load ML models on startup"""
    model_path = Path(os.getenv("MODEL_PATH", "./models"))
    
    try:
        models["risk"] = joblib.load(model_path / "risk_model.pkl")
        models["anomaly"] = joblib.load(model_path / "anomaly_model.pkl")
        models["burnout"] = joblib.load(model_path / "burnout_model.pkl")
        print("All models loaded successfully")
    except FileNotFoundError as e:
        print(f"Warning: Model file not found - {e}")
        print("Using default models...")
        # Initialize default models
        from sklearn.ensemble import RandomForestClassifier
        models["risk"] = RandomForestClassifier()
        models["anomaly"] = RandomForestClassifier()
        models["burnout"] = RandomForestClassifier()
```

### CORS Issues

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');

const app = express();

// Configure CORS
const corsOptions = {
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));

// Handle preflight requests
app.options('*', cors(corsOptions));
```

### Performance Optimization for Large Datasets

```javascript
// backend
