---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement task tracking with AI insights"
  - "create admin dashboard with ML predictions"
  - "build ticket classification system"
  - "add risk detection to user management"
  - "deploy user management with AI features"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform combining React frontend, Node.js backend, and FastAPI ML service. Features role-based access control, Kanban task boards, support ticket management, and AI-powered analytics including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB 4.4+

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
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MODEL_PATH=./models
LOG_LEVEL=info
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

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

## Architecture Overview

### Three-Tier Architecture

1. **Frontend (React)**: User interface with role-based dashboards
2. **Backend (Node.js)**: REST APIs, authentication, business logic
3. **ML Service (FastAPI)**: AI models for predictions and analytics

## Backend API Reference

### Authentication

```javascript
// Register new user
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

// Login
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

// Authenticated requests
const getAuthHeaders = () => ({
  'Content-Type': 'application/json',
  'Authorization': `Bearer ${localStorage.getItem('token')}`
});
```

### User Management (Admin)

```javascript
// Get all users
const getAllUsers = async () => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: getAuthHeaders()
  });
  return response.json();
};

// Update user
const updateUser = async (userId, updates) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: getAuthHeaders(),
    body: JSON.stringify(updates)
  });
  return response.json();
};

// Delete user
const deleteUser = async (userId) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: getAuthHeaders()
  });
  return response.json();
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: getAuthHeaders(),
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'inprogress', 'done'
      deadline: taskData.deadline
    })
  });
  return response.json();
};

// Get user tasks
const getUserTasks = async (userId) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: getAuthHeaders()
  });
  return response.json();
};

// Update task status
const updateTaskStatus = async (taskId, status) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: getAuthHeaders(),
    body: JSON.stringify({ status })
  });
  return response.json();
};

// Track time
const startTimeTracking = async (taskId) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time/start`, {
    method: 'POST',
    headers: getAuthHeaders()
  });
  return response.json();
};

const stopTimeTracking = async (taskId) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time/stop`, {
    method: 'POST',
    headers: getAuthHeaders()
  });
  return response.json();
};
```

### Support Tickets

```javascript
// Create ticket
const createTicket = async (ticketData) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: getAuthHeaders(),
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category // 'technical', 'hr', 'access', etc.
    })
  });
  return response.json();
};

// Get tickets
const getTickets = async (filters = {}) => {
  const params = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tickets?${params}`, {
    headers: getAuthHeaders()
  });
  return response.json();
};

// Update ticket
const updateTicket = async (ticketId, updates) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: getAuthHeaders(),
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

## ML Service API Reference

### AI-Powered Ticket Classification

```javascript
// Classify and route ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      title: "Support request title"
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', assignedDepartment: 'IT' }
  return result;
};
```

### Risk Prediction

```javascript
// Predict user risk score
const predictUserRisk = async (userId, userMetrics) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      failedLogins: userMetrics.failedLogins,
      unusualActivity: userMetrics.unusualActivity,
      accessPatterns: userMetrics.accessPatterns,
      taskCompletionRate: userMetrics.taskCompletionRate
    })
  });
  const result = await response.json();
  // Returns: { riskScore: 0.75, riskLevel: 'high', factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// Detect anomalous behavior
const detectAnomaly = async (behaviorData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: behaviorData.userId,
      loginTime: behaviorData.loginTime,
      location: behaviorData.location,
      deviceInfo: behaviorData.deviceInfo,
      activityPattern: behaviorData.activityPattern
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.85, reasons: [...] }
  return result;
};
```

### Burnout Analysis

```javascript
// Analyze employee burnout risk
const analyzeBurnout = async (userId, workloadData) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-analysis', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      hoursWorked: workloadData.hoursWorked,
      tasksCompleted: workloadData.tasksCompleted,
      overtimeHours: workloadData.overtimeHours,
      weekendWork: workloadData.weekendWork,
      taskDensity: workloadData.taskDensity
    })
  });
  const result = await response.json();
  // Returns: { burnoutScore: 0.6, level: 'moderate', recommendations: [...] }
  return result;
};
```

### Predictive Project Insights

```javascript
// Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/project-prediction', {
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
  const result = await response.json();
  // Returns: { delayProbability: 0.45, estimatedDelay: 5, suggestions: [...] }
  return result;
};
```

## Frontend Components

### React Context for Authentication

```javascript
// AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      // Verify token and get user data
      fetchUserProfile();
    }
  }, [token]);

  const fetchUserProfile = async () => {
    try {
      const response = await fetch('http://localhost:5000/api/auth/profile', {
        headers: {
          'Authorization': `Bearer ${token}`
        }
      });
      const data = await response.json();
      setUser(data);
    } catch (error) {
      logout();
    }
  };

  const login = async (email, password) => {
    const response = await fetch('http://localhost:5000/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    setToken(data.token);
    setUser(data.user);
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
// KanbanBoard.jsx
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inprogress: [],
    done: []
  });

  useEffect(() => {
    loadTasks();
  }, [userId]);

  const loadTasks = async () => {
    const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
      headers: {
        'Authorization': `Bearer ${localStorage.getItem('token')}`
      }
    });
    const data = await response.json();
    
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inprogress: data.filter(t => t.status === 'inprogress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${localStorage.getItem('token')}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    loadTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inprogress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority ${task.priority}`}>
                {task.priority}
              </span>
              <div className="task-actions">
                {status !== 'done' && (
                  <button onClick={() => moveTask(task._id, 
                    status === 'todo' ? 'inprogress' : 'done')}>
                    Move →
                  </button>
                )}
              </div>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Admin Dashboard with AI Insights

```javascript
// AdminDashboard.jsx
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskAlerts, setRiskAlerts] = useState([]);
  const [burnoutAlerts, setBurnoutAlerts] = useState([]);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    const token = localStorage.getItem('token');
    
    // Load users
    const usersResponse = await fetch('http://localhost:5000/api/users', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const usersData = await usersResponse.json();
    setUsers(usersData);

    // Get AI insights for each user
    for (const user of usersData) {
      // Risk analysis
      const riskResponse = await fetch('http://localhost:8000/api/ml/risk-prediction', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          userId: user._id,
          failedLogins: user.metrics?.failedLogins || 0,
          unusualActivity: user.metrics?.unusualActivity || false,
          taskCompletionRate: user.metrics?.taskCompletionRate || 1
        })
      });
      const riskData = await riskResponse.json();
      
      if (riskData.riskLevel === 'high') {
        setRiskAlerts(prev => [...prev, { user: user.name, ...riskData }]);
      }

      // Burnout analysis
      const burnoutResponse = await fetch('http://localhost:8000/api/ml/burnout-analysis', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          userId: user._id,
          hoursWorked: user.metrics?.hoursWorked || 40,
          tasksCompleted: user.metrics?.tasksCompleted || 0,
          overtimeHours: user.metrics?.overtimeHours || 0
        })
      });
      const burnoutData = await burnoutResponse.json();
      
      if (burnoutData.level === 'high') {
        setBurnoutAlerts(prev => [...prev, { user: user.name, ...burnoutData }]);
      }
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <section className="alerts">
        <div className="alert-card risk">
          <h2>🚨 Risk Alerts</h2>
          {riskAlerts.map((alert, i) => (
            <div key={i} className="alert-item">
              <strong>{alert.user}</strong>
              <p>Risk Score: {alert.riskScore.toFixed(2)}</p>
              <p>Factors: {alert.factors?.join(', ')}</p>
            </div>
          ))}
        </div>

        <div className="alert-card burnout">
          <h2>⚠️ Burnout Alerts</h2>
          {burnoutAlerts.map((alert, i) => (
            <div key={i} className="alert-item">
              <strong>{alert.user}</strong>
              <p>Burnout Score: {alert.burnoutScore.toFixed(2)}</p>
              <p>{alert.recommendations?.join(', ')}</p>
            </div>
          ))}
        </div>
      </section>

      <section className="users-table">
        <h2>Users</h2>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.name}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.isActive ? 'Active' : 'Inactive'}</td>
                <td>
                  <button onClick={() => editUser(user._id)}>Edit</button>
                  <button onClick={() => deleteUser(user._id)}>Delete</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>
    </div>
  );
};

export default AdminDashboard;
```

## Configuration

### Backend Environment Variables

```env
# Server
PORT=5000
NODE_ENV=development

# Database
MONGO_URI=mongodb://localhost:27017/enterprise_user_mgmt
MONGO_OPTIONS={"useNewUrlParser": true, "useUnifiedTopology": true}

# Authentication
JWT_SECRET=your_secure_jwt_secret_key
JWT_EXPIRE=7d
BCRYPT_ROUNDS=10

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (for notifications)
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=noreply@example.com
SMTP_PASS=your_smtp_password

# Security
RATE_LIMIT_WINDOW=15m
RATE_LIMIT_MAX_REQUESTS=100
CORS_ORIGIN=http://localhost:3000
```

### ML Service Configuration

```python
# ml-service/config.py
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    # Model paths
    MODEL_PATH: str = "./models"
    TICKET_CLASSIFIER_PATH: str = "./models/ticket_classifier.pkl"
    RISK_MODEL_PATH: str = "./models/risk_predictor.pkl"
    
    # Model parameters
    RISK_THRESHOLD: float = 0.7
    BURNOUT_THRESHOLD: float = 0.6
    ANOMALY_THRESHOLD: float = 0.8
    
    # Logging
    LOG_LEVEL: str = "info"
    
    # API
    API_KEY: str = os.getenv("ML_API_KEY", "")
    
    class Config:
        env_file = ".env"

settings = Settings()
```

### Frontend Environment Variables

```env
# API Endpoints
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000

# Feature Flags
REACT_APP_ENABLE_AI_FEATURES=true
REACT_APP_ENABLE_TIME_TRACKING=true

# UI
REACT_APP_THEME=light
REACT_APP_DASHBOARD_REFRESH_INTERVAL=60000
```

## Common Patterns

### Protected Routes

```javascript
// ProtectedRoute.jsx
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from './AuthContext';

const ProtectedRoute = ({ children, requiredRole }) => {
  const { user, token } = useContext(AuthContext);

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && user?.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Routes>
          <Route path="/login" element={<Login />} />
          <Route path="/user-dashboard" element={
            <ProtectedRoute>
              <UserDashboard />
            </ProtectedRoute>
          } />
          <Route path="/admin-dashboard" element={
            <ProtectedRoute requiredRole="admin">
              <AdminDashboard />
            </ProtectedRoute>
          } />
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}
```

### Real-time Notifications

```javascript
// useNotifications.js
import { useState, useEffect } from 'react';

export const useNotifications = (userId) => {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    const eventSource = new EventSource(
      `http://localhost:5000/api/notifications/stream/${userId}`
    );

    eventSource.onmessage = (event) => {
      const notification = JSON.parse(event.data);
      setNotifications(prev => [notification, ...prev]);
    };

    return () => eventSource.close();
  }, [userId]);

  const markAsRead = async (notificationId) => {
    await fetch(`http://localhost:5000/api/notifications/${notificationId}/read`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${localStorage.getItem('token')}`
      }
    });
    setNotifications(prev => 
      prev.map(n => n._id === notificationId ? { ...n, read: true } : n)
    );
  };

  return { notifications, markAsRead };
};
```

### Batch User Operations

```javascript
// Bulk user updates for admins
const bulkUpdateUsers = async (userIds, updates) => {
  const response = await fetch('http://localhost:5000/api/users/bulk', {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${localStorage.getItem('token')}`
    },
    body: JSON.stringify({
      userIds,
      updates
    })
  });
  return response.json();
};

// Usage
await bulkUpdateUsers(
  ['user1', 'user2', 'user3'],
  { department: 'Engineering', status: 'active' }
);
```

### AI-Enhanced Search

```javascript
// Intelligent ticket search with ML
const searchTickets = async (query) => {
  // First classify the query
  const classification = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ text: query })
  }).then(r => r.json());

  // Then search with enhanced filters
  const response = await fetch('http://localhost:5000/api/tickets/search', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${localStorage.getItem('token')}`
    },
    body: JSON.stringify({
      query,
      category: classification.category,
      priority: classification.priority
    })
  });
  
  return response.json();
};
```

## Troubleshooting

### Common Issues

**JWT Token Expiration**
```javascript
// Implement token refresh
const refreshToken = async () => {
  try {
    const response = await fetch('http://localhost:5000/api/auth/refresh', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${localStorage.getItem('token')}`
      }
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    return data.token;
  } catch (error) {
    // Redirect to login
    window.location.href = '/login';
  }
};

// Use axios interceptor for automatic refresh
import axios from 'axios';

axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      const newToken = await refreshToken();
      error.config.headers['Authorization'] = `Bearer ${newToken}`;
      return axios.request(error.config);
    }
    return Promise.reject(error);
  }
);
```

**MongoDB Connection Issues**
```javascript
// Backend connection with retry logic
const mongoose = require('mongoose');

const connectDB = async (retries = 5) => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected');
  } catch (error) {
    if (retries > 0) {
      console.log(`Retrying connection... (${retries} attempts left)`);
      setTimeout(() => connectDB(retries - 1), 5000);
    } else {
      console.error('MongoDB connection failed:', error);
      process.exit(1);
    }
  }
};
```

**ML Model Loading Errors**
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
import pickle
import os

app = FastAPI()

models = {}

@app.on_event("startup")
async def load_models():
    try:
        model_path = os.getenv("MODEL_PATH", "./models")
        
        if os.path.exists(f"{model_path}/ticket_classifier.pkl"):
            with open(f"{model_path}/ticket_classifier.pkl", "rb") as f:
                models["ticket_classifier"] = pickle.load(f)
        else:
            print("Warning: Ticket classifier not found, using fallback")
            models["ticket_classifier"] = None
            
        if os.path.exists(f"{model_path}/risk_predictor.pkl"):
            with open(f"{model_path}/risk_predictor.pkl", "rb") as f:
                models["risk_predictor"] = pickle.load(f)
        else:
            print("Warning: Risk predictor not found, using fallback")
            models["risk_predictor"] = None
            
    except Exception as e:
        print(f"Error loading models: {e}")
```

**CORS Issues**
```javascript
// Backend CORS configuration
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

**Performance Optimization**
```javascript
// Implement caching for frequently accessed data
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 300 }); // 5 minutes

const getCachedUsers = async () => {
  const cacheKey = 'all_users';
  let users = cache.get(cacheKey);
  
  if (!users) {
    users = await User.find({}).select('-password');
    cache.set(cacheKey, users);
  }
  
  return users;
};

// Clear cache on updates
const updateUser = async (userId, updates) => {
  const user = await User.findByIdAndUpdate(userId, updates, { new: true });
  cache.del('all_users'); // Invalidate cache
  return user;
};
```

## Database Schema

### User Model
```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  department: String,
  isActive: { type: Boolean, default: true },
  lastLogin: Date,
  metrics: {
    failedLogins: { type: Number, default: 0 },
    taskCompletionRate: { type: Number, default: 1.0 },
    hoursWorked: { type: Number, default: 0 },
    overtimeHours: { type: Number, default: 0 }
  },
  createdAt: { type
