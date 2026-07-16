---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, task management, and automated ticket routing
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user task tracking with AI insights"
  - "build admin dashboard with ML predictions"
  - "integrate AI ticket classification system"
  - "add burnout detection and anomaly detection"
  - "configure JWT authentication for user management"
  - "deploy user management system with FastAPI ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack web application for managing users, tasks, and support tickets with integrated AI analytics. The system provides smart insights including risk detection, anomaly detection, burnout analysis, and predictive project insights to improve organizational productivity.

## What This Project Does

This enterprise system provides:
- **User Management**: Secure authentication, role-based access control, and user CRUD operations
- **Task Management**: Kanban boards, time tracking, and task assignment
- **Support Tickets**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Admin Dashboard**: Organization-wide analytics, audit logs, and alerts

## Architecture

The project consists of three main components:
1. **Frontend** (React.js) - User interface and admin dashboard
2. **Backend** (Node.js) - REST APIs, authentication, business logic
3. **ML Service** (FastAPI + scikit-learn) - AI models and predictions

## Installation

### Prerequisites
- Node.js 14+
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
MONGODB_URI=mongodb://localhost:27017/enterprise_users
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
MONGODB_URI=mongodb://localhost:27017/enterprise_users
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

Create `.env` file:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Backend API Usage

### Authentication

```javascript
// User Registration
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: 'user' // or 'admin'
    })
  });
  return await response.json();
};

// User Login
const loginUser = async (email, password) => {
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

// Protected API Call with JWT
const fetchUserProfile = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users/profile', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

### User Management (Admin)

```javascript
// Get All Users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Create User
const createUser = async (userData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(userData)
  });
  return await response.json();
};

// Update User
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
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

// Delete User
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### Task Management

```javascript
// Get User Tasks
const getUserTasks = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Create Task
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
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
      status: 'todo' // 'todo', 'inprogress', 'done'
    })
  });
  return await response.json();
};

// Update Task Status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return await response.json();
};

// Track Time on Task
const trackTime = async (taskId, timeSpent) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ timeSpent }) // in minutes
  });
  return await response.json();
};
```

### Support Tickets

```javascript
// Create Support Ticket
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
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
      category: ticketData.category // 'technical', 'access', 'general'
    })
  });
  return await response.json();
};

// Get User Tickets
const getUserTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Update Ticket Status (Admin)
const updateTicketStatus = async (ticketId, status) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'open', 'in_progress', 'resolved'
  });
  return await response.json();
};
```

## ML Service API Usage

### AI Ticket Classification

```javascript
// Classify Ticket with AI
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: "Need help with login"
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.87 }
  return data;
};
```

### Risk Prediction

```javascript
// Predict User Risk Score
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      features: {
        taskCompletionRate: 0.65,
        averageTaskDelay: 2.5, // days
        loginFrequency: 15, // per month
        ticketsRaised: 8,
        workloadHours: 45
      }
    })
  });
  const data = await response.json();
  // Returns: { riskScore: 0.72, riskLevel: 'high', factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// Detect Anomalies in User Behavior
const detectAnomalies = async (userActivity) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userActivity.userId,
      activities: [
        { timestamp: '2026-01-20T10:30:00Z', action: 'login', ip: '192.168.1.1' },
        { timestamp: '2026-01-20T22:00:00Z', action: 'login', ip: '10.0.0.5' },
        { timestamp: '2026-01-20T22:15:00Z', action: 'data_export', recordCount: 10000 }
      ]
    })
  });
  const data = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.89, alerts: [...] }
  return data;
};
```

### Burnout Detection

```javascript
// Analyze User Burnout Risk
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      metrics: {
        weeklyWorkHours: 55,
        overtimeHours: 15,
        tasksCompleted: 45,
        missedDeadlines: 8,
        stressIndicators: 7 // 0-10 scale
      }
    })
  });
  const data = await response.json();
  // Returns: { burnoutRisk: 'high', score: 0.78, recommendations: [...] }
  return data;
};
```

### Project Delay Prediction

```javascript
// Predict Project Completion Delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/project-insights', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.id,
      tasksTotal: 50,
      tasksCompleted: 20,
      daysElapsed: 30,
      daysRemaining: 20,
      teamSize: 5,
      complexityScore: 7.5
    })
  });
  const data = await response.json();
  // Returns: { delayPrediction: 10, confidence: 0.82, suggestions: [...] }
  return data;
};
```

## React Component Examples

### User Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [tickets, setTickets] = useState([]);
  const [burnoutRisk, setBurnoutRisk] = useState(null);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    const token = localStorage.getItem('token');
    
    // Fetch tasks
    const tasksRes = await fetch('http://localhost:5000/api/tasks', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const tasksData = await tasksRes.json();
    setTasks(tasksData);

    // Fetch tickets
    const ticketsRes = await fetch('http://localhost:5000/api/tickets', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const ticketsData = await ticketsRes.json();
    setTickets(ticketsData);

    // Check burnout risk
    const burnoutRes = await fetch('http://localhost:8000/api/ml/burnout-detection', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        userId: localStorage.getItem('userId'),
        metrics: calculateUserMetrics(tasksData)
      })
    });
    const burnoutData = await burnoutRes.json();
    setBurnoutRisk(burnoutData);
  };

  const calculateUserMetrics = (tasks) => {
    const completed = tasks.filter(t => t.status === 'done').length;
    const missed = tasks.filter(t => new Date(t.dueDate) < new Date() && t.status !== 'done').length;
    
    return {
      weeklyWorkHours: 40,
      overtimeHours: 5,
      tasksCompleted: completed,
      missedDeadlines: missed,
      stressIndicators: missed > 5 ? 8 : 4
    };
  };

  return (
    <div className="dashboard">
      <h1>User Dashboard</h1>
      
      {burnoutRisk && burnoutRisk.burnoutRisk === 'high' && (
        <div className="alert alert-warning">
          ⚠️ High burnout risk detected! Consider reducing workload.
        </div>
      )}

      <div className="stats">
        <div className="stat-card">
          <h3>Tasks</h3>
          <p>{tasks.length}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p>{tickets.filter(t => t.status === 'open').length}</p>
        </div>
      </div>

      <KanbanBoard tasks={tasks} setTasks={setTasks} />
    </div>
  );
};

export default UserDashboard;
```

### Kanban Board Component

```javascript
import React from 'react';

const KanbanBoard = ({ tasks, setTasks }) => {
  const moveTask = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    
    if (response.ok) {
      setTasks(tasks.map(task => 
        task._id === taskId ? { ...task, status: newStatus } : task
      ));
    }
  };

  const columns = {
    todo: tasks.filter(t => t.status === 'todo'),
    inprogress: tasks.filter(t => t.status === 'inprogress'),
    done: tasks.filter(t => t.status === 'done')
  };

  return (
    <div className="kanban-board">
      {Object.entries(columns).map(([status, columnTasks]) => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {columnTasks.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status}
                onChange={(e) => moveTask(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="inprogress">In Progress</option>
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

### Admin Analytics Dashboard

```javascript
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskUsers, setRiskUsers] = useState([]);
  const [anomalies, setAnomalies] = useState([]);

  useEffect(() => {
    fetchAdminData();
  }, []);

  const fetchAdminData = async () => {
    const token = localStorage.getItem('token');
    
    // Fetch all users
    const usersRes = await fetch('http://localhost:5000/api/users', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const usersData = await usersRes.json();
    setUsers(usersData);

    // Get risk predictions for all users
    const riskPromises = usersData.map(async (user) => {
      const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          userId: user._id,
          features: calculateUserFeatures(user)
        })
      });
      const riskData = await response.json();
      return { ...user, risk: riskData };
    });

    const risks = await Promise.all(riskPromises);
    setRiskUsers(risks.filter(u => u.risk.riskLevel === 'high'));
  };

  const calculateUserFeatures = (user) => {
    return {
      taskCompletionRate: user.stats?.completionRate || 0.7,
      averageTaskDelay: user.stats?.avgDelay || 1.5,
      loginFrequency: user.stats?.loginFreq || 20,
      ticketsRaised: user.stats?.tickets || 3,
      workloadHours: user.stats?.hours || 40
    };
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p>{users.length}</p>
        </div>
        <div className="stat-card danger">
          <h3>High Risk Users</h3>
          <p>{riskUsers.length}</p>
        </div>
      </div>

      <div className="risk-alerts">
        <h2>High Risk Users</h2>
        {riskUsers.map(user => (
          <div key={user._id} className="alert-card">
            <h4>{user.name}</h4>
            <p>Risk Score: {(user.risk.riskScore * 100).toFixed(0)}%</p>
            <ul>
              {user.risk.factors?.map((factor, i) => (
                <li key={i}>{factor}</li>
              ))}
            </ul>
          </div>
        ))}
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
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_users

# Authentication
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRES_IN=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@example.com
SMTP_PASS=your_app_password

# Logging
LOG_LEVEL=info
```

### ML Service Environment Variables

```env
# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_users

# Models
MODEL_PATH=./models
RETRAIN_INTERVAL=86400

# ML Settings
RISK_THRESHOLD=0.7
ANOMALY_THRESHOLD=0.8
BURNOUT_THRESHOLD=0.75

# Logging
LOG_LEVEL=INFO
```

### Frontend Environment Variables

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENV=development
```

## Common Patterns

### Protected Route with Role-Based Access

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const userRole = localStorage.getItem('role');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && userRole !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

// Usage
<Route path="/admin" element={
  <ProtectedRoute requiredRole="admin">
    <AdminDashboard />
  </ProtectedRoute>
} />
```

### Real-time Task Timer

```javascript
import React, { useState, useEffect } from 'react';

const TaskTimer = ({ taskId }) => {
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

  const saveTime = async () => {
    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ timeSpent: Math.floor(seconds / 60) })
    });
    setSeconds(0);
    setIsRunning(false);
  };

  return (
    <div className="task-timer">
      <p>{Math.floor(seconds / 3600)}h {Math.floor((seconds % 3600) / 60)}m {seconds % 60}s</p>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      {seconds > 0 && <button onClick={saveTime}>Save</button>}
    </div>
  );
};

export default TaskTimer;
```

### Batch User Import

```javascript
const importUsers = async (csvFile) => {
  const token = localStorage.getItem('token');
  const formData = new FormData();
  formData.append('file', csvFile);

  const response = await fetch('http://localhost:5000/api/users/import', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`
    },
    body: formData
  });

  const result = await response.json();
  // Returns: { imported: 45, failed: 2, errors: [...] }
  return result;
};
```

## Troubleshooting

### JWT Token Expired

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

// Interceptor for expired token
const fetchWithAuth = async (url, options = {}) => {
  let token = localStorage.getItem('token');
  let response = await fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`
    }
  });

  if (response.status === 401) {
    token = await refreshToken();
    response = await fetch(url, {
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
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Model Not Loading

```python
# ml-service/models/loader.py
import os
import pickle
from pathlib import Path

def load_model(model_name):
    model_path = Path(os.getenv('MODEL_PATH', './models')) / f'{model_name}.pkl'
    
    if not model_path.exists():
        print(f"Model {model_name} not found, training new model...")
        return train_new_model(model_name)
    
    try:
        with open(model_path, 'rb') as f:
            model = pickle.load(f)
        return model
    except Exception as e:
        print(f"Error loading model: {e}")
        return train_new_model(model_name)
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

const corsOptions = {
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));
```

### Performance Optimization for Large User Base

```javascript
// Implement pagination
const getUsersWithPagination = async (page = 1, limit = 50) => {
  const token = localStorage.getItem('token');
  const response = await fetch(
    `http://localhost:5000/api/users?page=${page}&limit=${limit}`,
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return await response.json();
  // Returns: { users: [...], total: 500, page: 1, pages: 10 }
};

// Implement caching for frequently accessed data
const cacheManager = {
  cache: new Map(),
  ttl: 5 * 60 * 1000, // 5 minutes

  get: function(key) {
    const item = this.cache.get(key);
    if (item && Date.now() < item.expiry) {
      return item.data;
    }
    this.cache.delete(key);
    return null;
  },

  set: function(key, data) {
    this.cache.set(key, {
      data,
      expiry: Date.now() + this.ttl
    });
  }
};
```

This skill provides comprehensive guidance for using the Enterprise User Management System with AI Analytics, covering setup, API usage, AI features, and common implementation patterns for both developers and AI coding agents.
