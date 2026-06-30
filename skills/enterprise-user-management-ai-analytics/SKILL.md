---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "integrate ML service for anomaly detection"
  - "build admin panel for user management"
  - "configure JWT authentication for enterprise app"
  - "add AI ticket classification and routing"
  - "implement burnout detection and risk prediction"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines user/task management with AI-powered analytics. It features role-based access control, Kanban-style task tracking, support ticket management, and ML-driven insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

**Stack**: React frontend, Node.js/Express backend, FastAPI ML service, MongoDB database, JWT authentication

## Installation

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
MONGODB_URI=your_mongodb_connection_string
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
BACKEND_URL=http://localhost:5000
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

Create `.env` file:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Backend API (Node.js/Express)

### Authentication Routes

```javascript
// User registration
const register = async (userData) => {
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
  return await response.json();
};

// User login
const login = async (credentials) => {
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

### User Management (Admin)

```javascript
// Get all users (admin only)
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};

// Update user
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
  return await response.json();
};

// Delete user
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
};
```

### Task Management

```javascript
// Create task
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
      status: 'todo', // 'todo', 'in-progress', 'done'
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate
    })
  });
  return await response.json();
};

// Get user tasks
const getUserTasks = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks/my-tasks', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};

// Update task status
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
  return await response.json();
};

// Track time on task
const trackTime = async (taskId, timeSpent) => {
  const token = localStorage.getItem('token');
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

### Support Tickets

```javascript
// Create ticket
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return await response.json();
};

// Get all tickets (admin)
const getAllTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};

// Update ticket status
const updateTicketStatus = async (ticketId, status) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status }) // 'open', 'in-progress', 'resolved', 'closed'
  });
  return await response.json();
};
```

## ML Service API (FastAPI)

### AI Ticket Classification

```javascript
// Classify and route ticket automatically
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      text: ticketText
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', suggested_assignee: 'user_id' }
  return result;
};
```

### Risk Prediction

```javascript
// Predict user risk based on behavior patterns
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId
    })
  });
  const result = await response.json();
  // Returns: { risk_score: 0.75, risk_level: 'high', factors: ['...'] }
  return result;
};
```

### Anomaly Detection

```javascript
// Detect anomalous user behavior
const detectAnomalies = async (userId, activityData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      login_time: activityData.loginTime,
      login_location: activityData.location,
      actions: activityData.actions,
      session_duration: activityData.duration
    })
  });
  const result = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.85, alert_level: 'medium' }
  return result;
};
```

### Burnout Detection

```javascript
// Analyze user workload for burnout risk
const analyzeBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/analyze-burnout', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId
    })
  });
  const result = await response.json();
  // Returns: { burnout_risk: 'high', workload_score: 8.5, recommendations: ['...'] }
  return result;
};
```

### Predictive Project Insights

```javascript
// Predict project completion and delays
const predictProjectInsights = async (projectId) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-project', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      project_id: projectId
    })
  });
  const result = await response.json();
  // Returns: { completion_probability: 0.65, expected_delay_days: 5, bottlenecks: ['...'] }
  return result;
};
```

## React Component Patterns

### Authentication Context

```javascript
import React, { createContext, useState, useContext, useEffect } from 'react';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      // Verify token and load user data
      fetchUserData();
    }
  }, [token]);

  const fetchUserData = async () => {
    try {
      const response = await fetch('http://localhost:5000/api/auth/me', {
        headers: {
          'Authorization': `Bearer ${token}`
        }
      });
      const data = await response.json();
      setUser(data.user);
    } catch (error) {
      logout();
    }
  };

  const login = async (credentials) => {
    const response = await fetch('http://localhost:5000/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(credentials)
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

export const useAuth = () => useContext(AuthContext);
```

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';
import { useAuth } from '../context/AuthContext';

const KanbanBoard = () => {
  const { token } = useAuth();
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch('http://localhost:5000/api/tasks/my-tasks', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    const categorized = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(categorized);
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority ${task.priority}`}>{task.priority}</span>
      <div className="task-actions">
        {task.status !== 'done' && (
          <button onClick={() => moveTask(task._id, 
            task.status === 'todo' ? 'in-progress' : 'done')}>
            Move →
          </button>
        )}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <div className="column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### Admin Dashboard with AI Insights

```javascript
import React, { useState, useEffect } from 'react';
import { useAuth } from '../context/AuthContext';

const AdminDashboard = () => {
  const { token } = useAuth();
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState({});
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    // Fetch users
    const usersRes = await fetch('http://localhost:5000/api/users', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const usersData = await usersRes.json();
    setUsers(usersData);

    // Fetch analytics
    const analyticsRes = await fetch('http://localhost:5000/api/analytics/overview', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const analyticsData = await analyticsRes.json();
    setAnalytics(analyticsData);

    // Check AI insights for each user
    for (const user of usersData) {
      checkUserInsights(user._id);
    }
  };

  const checkUserInsights = async (userId) => {
    // Check burnout
    const burnoutRes = await fetch('http://localhost:8000/api/ml/analyze-burnout', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ user_id: userId })
    });
    const burnout = await burnoutRes.json();

    if (burnout.burnout_risk === 'high') {
      setAlerts(prev => [...prev, {
        type: 'burnout',
        userId,
        message: `User at high burnout risk (score: ${burnout.workload_score})`
      }]);
    }

    // Check anomalies
    const anomalyRes = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ user_id: userId })
    });
    const anomaly = await anomalyRes.json();

    if (anomaly.is_anomaly) {
      setAlerts(prev => [...prev, {
        type: 'anomaly',
        userId,
        message: `Anomalous behavior detected (score: ${anomaly.anomaly_score})`
      }]);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-value">{users.length}</p>
        </div>
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p className="stat-value">{analytics.activeTasks || 0}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p className="stat-value">{analytics.openTickets || 0}</p>
        </div>
        <div className="stat-card alert">
          <h3>AI Alerts</h3>
          <p className="stat-value">{alerts.length}</p>
        </div>
      </div>

      {alerts.length > 0 && (
        <div className="alerts-section">
          <h2>AI-Powered Alerts</h2>
          {alerts.map((alert, idx) => (
            <div key={idx} className={`alert-card ${alert.type}`}>
              <span className="alert-icon">⚠️</span>
              <p>{alert.message}</p>
            </div>
          ))}
        </div>
      )}

      <div className="users-table">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Tasks</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.name}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.taskCount || 0}</td>
                <td>
                  <button onClick={() => editUser(user._id)}>Edit</button>
                  <button onClick={() => deleteUser(user._id)}>Delete</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
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
NODE_ENV=production

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
MONGODB_OPTIONS=retryWrites=true&w=majority

# Authentication
JWT_SECRET=your_secure_jwt_secret_key
JWT_EXPIRE=7d
BCRYPT_ROUNDS=10

# ML Service
ML_SERVICE_URL=http://localhost:8000
ML_SERVICE_TIMEOUT=30000

# Email (for notifications)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password

# Logging
LOG_LEVEL=info
```

### ML Service Configuration

```env
# Model Settings
MODEL_PATH=./models
MODEL_RETRAIN_INTERVAL=86400
MIN_TRAINING_SAMPLES=100

# API
BACKEND_URL=http://localhost:5000
API_TIMEOUT=60

# ML Parameters
ANOMALY_THRESHOLD=0.7
RISK_THRESHOLD=0.6
BURNOUT_THRESHOLD=7.0

# Logging
LOG_LEVEL=INFO
LOG_FILE=./logs/ml_service.log
```

### Frontend Environment Variables

```env
# API Endpoints
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000

# Features
REACT_APP_ENABLE_AI_FEATURES=true
REACT_APP_ENABLE_ANALYTICS=true

# UI
REACT_APP_THEME=default
REACT_APP_PAGINATION_SIZE=10
```

## Common Patterns

### Protected Routes

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, token } = useAuth();

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

// Usage in routes
<Route path="/admin" element={
  <ProtectedRoute adminOnly={true}>
    <AdminDashboard />
  </ProtectedRoute>
} />
```

### AI-Enhanced Ticket Creation

```javascript
const SmartTicketForm = () => {
  const { token } = useAuth();
  const [formData, setFormData] = useState({ title: '', description: '' });
  const [aiSuggestions, setAiSuggestions] = useState(null);

  const handleDescriptionChange = async (description) => {
    setFormData({ ...formData, description });

    // Get AI classification as user types
    if (description.length > 20) {
      const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ text: description })
      });
      const suggestions = await response.json();
      setAiSuggestions(suggestions);
    }
  };

  const submitTicket = async () => {
    const ticketData = {
      ...formData,
      category: aiSuggestions?.category,
      priority: aiSuggestions?.priority,
      suggestedAssignee: aiSuggestions?.suggested_assignee
    };

    await fetch('http://localhost:5000/api/tickets', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify(ticketData)
    });
  };

  return (
    <form onSubmit={(e) => { e.preventDefault(); submitTicket(); }}>
      <input
        type="text"
        placeholder="Title"
        value={formData.title}
        onChange={(e) => setFormData({ ...formData, title: e.target.value })}
      />
      <textarea
        placeholder="Description"
        value={formData.description}
        onChange={(e) => handleDescriptionChange(e.target.value)}
      />
      {aiSuggestions && (
        <div className="ai-suggestions">
          <p>AI Suggestions:</p>
          <span>Category: {aiSuggestions.category}</span>
          <span>Priority: {aiSuggestions.priority}</span>
        </div>
      )}
      <button type="submit">Create Ticket</button>
    </form>
  );
};
```

### Time Tracking Component

```javascript
import React, { useState, useEffect, useRef } from 'react';

const TimeTracker = ({ taskId, onTimeLogged }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [seconds, setSeconds] = useState(0);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else {
      clearInterval(intervalRef.current);
    }
    return () => clearInterval(intervalRef.current);
  }, [isRunning]);

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const remainingSeconds = secs % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
  };

  const logTime = async () => {
    const minutes = Math.floor(seconds / 60);
    const token = localStorage.getItem('token');
    
    await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ timeSpent: minutes })
    });

    setSeconds(0);
    setIsRunning(false);
    onTimeLogged(minutes);
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <div className="timer-controls">
        <button onClick={() => setIsRunning(!isRunning)}>
          {isRunning ? 'Pause' : 'Start'}
        </button>
        <button onClick={() => setSeconds(0)} disabled={isRunning}>
          Reset
        </button>
        <button onClick={logTime} disabled={isRunning || seconds === 0}>
          Log Time
        </button>
      </div>
    </div>
  );
};
```

## Troubleshooting

### Authentication Issues

**Problem**: "Token expired" or "Invalid token" errors

```javascript
// Add token refresh mechanism
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

// Add interceptor for API calls
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

**Problem**: Cannot connect to MongoDB

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    // Retry connection
    setTimeout(connectDB, 5000);
  }
};

mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected. Attempting to reconnect...');
  connectDB();
});

module.exports = connectDB;
```

### ML Service Not Responding

**Problem**: ML predictions timing out or failing

```javascript
// Add timeout and fallback for ML calls
const callMLServiceWithFallback = async (endpoint, data, fallback) => {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), 10000); // 10s timeout

  try {
    const response = await fetch(`http://localhost:8000/api/ml/${endpoint}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
      signal: controller.signal
    });
    clearTimeout(timeoutId);
    return await response.json();
  } catch (error) {
    clearTimeout(timeoutId);
    console.warn(`ML service call failed, using fallback: ${error.message}`);
    return fallback;
  }
};

// Usage
const ticketClassification = await callMLServiceWithFallback(
  'classify-ticket',
  { text: ticketDescription },
  { category: 'general', priority: 'medium', suggested_assignee: null }
);
```

### CORS Issues

**Problem**: CORS errors when frontend calls backend

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### ML Model Not Loading

**Problem**: ML models fail to load on service startup

```python
# ml-
