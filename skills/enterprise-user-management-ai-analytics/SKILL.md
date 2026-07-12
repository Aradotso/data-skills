---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement AI-powered user management system"
  - "create admin dashboard with ML insights"
  - "build task management with anomaly detection"
  - "integrate AI ticket classification system"
  - "deploy user management with burnout analysis"
  - "configure JWT authentication for enterprise app"
  - "add predictive analytics to user dashboard"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack JavaScript application that combines user/task management with AI-powered analytics including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing. Built with React, Node.js, MongoDB, and FastAPI ML services.

## What This Project Does

This system provides:
- **User Management**: Role-based access control (Admin/User) with JWT authentication
- **Task Management**: Kanban board with time tracking and progress monitoring
- **Support Ticketing**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay forecasting
- **Admin Dashboard**: Organization-wide analytics, audit logs, and user monitoring
- **User Dashboard**: Personal task overview, performance metrics, and notifications

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+ and pip
- MongoDB (local or Atlas)

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

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key_here
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

Create `.env` file in ml-service directory:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
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

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Architecture Overview

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│   React     │─────▶│   Node.js    │─────▶│   MongoDB   │
│  Frontend   │      │   Backend    │      │  Database   │
└─────────────┘      └──────────────┘      └─────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │   FastAPI    │
                     │  ML Service  │
                     └──────────────┘
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
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role // 'admin' or 'user'
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
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management Endpoints

```javascript
// Get all users (Admin only)
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

// Delete user (Admin only)
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
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
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo, // user ID
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

// Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Track time on task
const trackTime = async (taskId, timeData, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      startTime: timeData.startTime,
      endTime: timeData.endTime,
      duration: timeData.duration // in seconds
    })
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
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category, // 'technical', 'billing', 'general'
      priority: ticketData.priority
    })
  });
  return await response.json();
};

// Get tickets with AI classification
const getTicketsWithAI = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets/classified', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

## ML Service API Usage

### Risk Prediction

```javascript
// Predict user risk score
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/ai/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      features: {
        login_frequency: 15,
        failed_login_attempts: 2,
        task_completion_rate: 0.85,
        avg_task_duration: 7200, // seconds
        tickets_raised: 3
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
// Detect anomalies in user behavior
const detectAnomalies = async (userActivity) => {
  const response = await fetch('http://localhost:8000/ai/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userActivity.userId,
      login_time: userActivity.loginTime,
      login_location: userActivity.loginLocation,
      device_info: userActivity.deviceInfo,
      actions_performed: userActivity.actions
    })
  });
  const data = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.87, details: {...} }
  return data;
};
```

### Burnout Analysis

```javascript
// Analyze user burnout risk
const analyzeBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/ai/burnout-analysis', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      metrics: {
        tasks_assigned: 25,
        tasks_completed: 18,
        avg_working_hours: 9.5,
        overtime_hours: 15,
        missed_deadlines: 3,
        stress_indicators: ['late_submissions', 'multiple_revisions']
      }
    })
  });
  const data = await response.json();
  // Returns: { burnout_risk: 'high', score: 0.78, recommendations: [...] }
  return data;
};
```

### Ticket Classification

```javascript
// AI-powered ticket classification
const classifyTicket = async (ticketContent) => {
  const response = await fetch('http://localhost:8000/ai/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketContent.title,
      description: ticketContent.description
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', suggested_assignee: 'team_id' }
  return data;
};
```

### Predictive Project Insights

```javascript
// Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/ai/project-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.id,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      days_remaining: projectData.daysRemaining,
      team_size: projectData.teamSize,
      avg_velocity: projectData.avgVelocity
    })
  });
  const data = await response.json();
  // Returns: { delay_probability: 0.65, estimated_delay_days: 5, suggestions: [...] }
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
      fetchUserData(token);
    }
  }, [token]);

  const fetchUserData = async (authToken) => {
    try {
      const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/me`, {
        headers: { 'Authorization': `Bearer ${authToken}` }
      });
      const userData = await response.json();
      setUser(userData);
    } catch (error) {
      console.error('Auth error:', error);
      logout();
    }
  };

  const login = async (email, password) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    setToken(data.token);
    setUser(data.user);
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return { user, token, login, logout, isAuthenticated: !!token };
};
```

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const KanbanBoard = () => {
  const { token } = useAuth();
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
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
    await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  const renderColumn = (title, status, taskList) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      <div className="task-list">
        {taskList.map(task => (
          <div key={task._id} className="task-card" draggable>
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority priority-${task.priority}`}>
              {task.priority}
            </span>
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
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('To Do', 'todo', tasks.todo)}
      {renderColumn('In Progress', 'in-progress', tasks.inProgress)}
      {renderColumn('Done', 'done', tasks.done)}
    </div>
  );
};

export default KanbanBoard;
```

### Admin Dashboard with AI Analytics

```javascript
// components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const AdminDashboard = () => {
  const { token } = useAuth();
  const [analytics, setAnalytics] = useState(null);
  const [users, setUsers] = useState([]);
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    // Fetch users
    const usersRes = await fetch(`${process.env.REACT_APP_API_URL}/users`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const usersData = await usersRes.json();
    setUsers(usersData);

    // Fetch analytics
    const analyticsRes = await fetch(`${process.env.REACT_APP_API_URL}/analytics/overview`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const analyticsData = await analyticsRes.json();
    setAnalytics(analyticsData);

    // Check for AI alerts
    await checkAIAlerts(usersData);
  };

  const checkAIAlerts = async (usersList) => {
    const alertsArr = [];
    
    for (const user of usersList) {
      // Check burnout risk
      const burnoutRes = await fetch(
        `${process.env.REACT_APP_ML_API_URL}/ai/burnout-analysis`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ user_id: user._id })
        }
      );
      const burnout = await burnoutRes.json();
      
      if (burnout.burnout_risk === 'high') {
        alertsArr.push({
          type: 'burnout',
          user: user.username,
          severity: 'high',
          message: `${user.username} is at high risk of burnout`
        });
      }

      // Check for anomalies
      const anomalyRes = await fetch(
        `${process.env.REACT_APP_ML_API_URL}/ai/anomaly-detection`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ user_id: user._id })
        }
      );
      const anomaly = await anomalyRes.json();
      
      if (anomaly.is_anomaly) {
        alertsArr.push({
          type: 'anomaly',
          user: user.username,
          severity: 'medium',
          message: `Unusual activity detected for ${user.username}`
        });
      }
    }
    
    setAlerts(alertsArr);
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      {/* AI Alerts Section */}
      <section className="alerts-section">
        <h2>AI Alerts</h2>
        {alerts.map((alert, idx) => (
          <div key={idx} className={`alert alert-${alert.severity}`}>
            <span className="alert-icon">⚠️</span>
            <span>{alert.message}</span>
          </div>
        ))}
      </section>

      {/* Analytics Overview */}
      {analytics && (
        <section className="analytics-overview">
          <div className="stat-card">
            <h3>Total Users</h3>
            <p>{analytics.totalUsers}</p>
          </div>
          <div className="stat-card">
            <h3>Active Tasks</h3>
            <p>{analytics.activeTasks}</p>
          </div>
          <div className="stat-card">
            <h3>Pending Tickets</h3>
            <p>{analytics.pendingTickets}</p>
          </div>
          <div className="stat-card">
            <h3>Completion Rate</h3>
            <p>{(analytics.completionRate * 100).toFixed(1)}%</p>
          </div>
        </section>
      )}

      {/* User Management Table */}
      <section className="user-management">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Username</th>
              <th>Email</th>
              <th>Role</th>
              <th>Tasks</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.username}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.taskCount || 0}</td>
                <td>
                  <span className={`status ${user.isActive ? 'active' : 'inactive'}`}>
                    {user.isActive ? 'Active' : 'Inactive'}
                  </span>
                </td>
                <td>
                  <button onClick={() => handleEditUser(user._id)}>Edit</button>
                  <button onClick={() => handleDeleteUser(user._id)}>Delete</button>
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

### Time Tracking Component

```javascript
// components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId, token }) => {
  const [isTracking, setIsTracking] = useState(false);
  const [startTime, setStartTime] = useState(null);
  const [elapsedTime, setElapsedTime] = useState(0);

  useEffect(() => {
    let interval;
    if (isTracking) {
      interval = setInterval(() => {
        setElapsedTime(Date.now() - startTime);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isTracking, startTime]);

  const handleStart = () => {
    setStartTime(Date.now());
    setIsTracking(true);
  };

  const handleStop = async () => {
    const endTime = Date.now();
    const duration = Math.floor((endTime - startTime) / 1000);
    
    await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({
        startTime: new Date(startTime),
        endTime: new Date(endTime),
        duration
      })
    });

    setIsTracking(false);
    setElapsedTime(0);
  };

  const formatTime = (ms) => {
    const seconds = Math.floor(ms / 1000);
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(elapsedTime)}</div>
      {!isTracking ? (
        <button onClick={handleStart} className="btn-start">Start</button>
      ) : (
        <button onClick={handleStop} className="btn-stop">Stop</button>
      )}
    </div>
  );
};

export default TimeTracker;
```

## Common Patterns

### Protected Routes

```javascript
// components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, requiredRole }) => {
  const { user, isAuthenticated } = useAuth();

  if (!isAuthenticated) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

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

### API Service Layer

```javascript
// services/api.js
const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

class ApiService {
  constructor() {
    this.token = localStorage.getItem('token');
  }

  setToken(token) {
    this.token = token;
    localStorage.setItem('token', token);
  }

  async request(endpoint, options = {}) {
    const headers = {
      'Content-Type': 'application/json',
      ...options.headers
    };

    if (this.token) {
      headers['Authorization'] = `Bearer ${this.token}`;
    }

    const response = await fetch(`${API_URL}${endpoint}`, {
      ...options,
      headers
    });

    if (!response.ok) {
      throw new Error(`API Error: ${response.statusText}`);
    }

    return await response.json();
  }

  // User methods
  async getUsers() {
    return this.request('/users');
  }

  async createUser(userData) {
    return this.request('/users', {
      method: 'POST',
      body: JSON.stringify(userData)
    });
  }

  // Task methods
  async getTasks(userId) {
    return this.request(`/tasks/user/${userId}`);
  }

  async updateTask(taskId, updates) {
    return this.request(`/tasks/${taskId}`, {
      method: 'PUT',
      body: JSON.stringify(updates)
    });
  }

  // AI methods
  async predictRisk(userId) {
    const response = await fetch(`${ML_API_URL}/ai/risk-prediction`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ user_id: userId })
    });
    return await response.json();
  }

  async analyzeBurnout(userId, metrics) {
    const response = await fetch(`${ML_API_URL}/ai/burnout-analysis`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ user_id: userId, metrics })
    });
    return await response.json();
  }
}

export default new ApiService();
```

## Configuration

### Backend Environment Variables

```env
# Server
PORT=5000
NODE_ENV=production

# Database
MONGODB_URI=mongodb+srv://user:password@cluster.mongodb.net/dbname

# Security
JWT_SECRET=your_strong_secret_key_min_32_chars
JWT_EXPIRE=7d
BCRYPT_ROUNDS=10

# ML Service
ML_SERVICE_URL=http://localhost:8000

# CORS
CORS_ORIGIN=http://localhost:3000

# Rate Limiting
RATE_LIMIT_WINDOW=15
RATE_LIMIT_MAX=100
```

### ML Service Configuration

```env
# MongoDB
MONGODB_URI=mongodb+srv://user:password@cluster.mongodb.net/dbname

# Model Settings
MODEL_PATH=./models
MODEL_UPDATE_INTERVAL=3600
PREDICTION_THRESHOLD=0.7

# Logging
LOG_LEVEL=INFO
LOG_FILE=./logs/ml-service.log

# Cache
REDIS_URL=redis://localhost:6379
CACHE_TTL=300
```

### Frontend Environment Variables

```env
# API Endpoints
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000

# App Settings
REACT_APP_NAME=Enterprise User Management
REACT_APP_VERSION=1.0.0

# Feature Flags
REACT_APP_ENABLE_AI_FEATURES=true
REACT_APP_ENABLE_NOTIFICATIONS=true
```

## Troubleshooting

### JWT Token Expired

```javascript
// Add token refresh logic
const refreshToken = async () => {
  try {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/refresh`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${localStorage.getItem('token')}`
      }
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    return data.token;
  } catch (error) {
    localStorage.removeItem('token');
    window.location.href = '/login';
  }
};
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
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    setTimeout(connectDB, 5000); // Retry after 5 seconds
  }
};

module.exports = connectDB;
```

### ML Service Not Responding

```python
# ml-service/main.py - Add health check
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI
