---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, and burnout analysis
triggers:
  - "setup enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement risk detection in user system"
  - "create user management dashboard with AI"
  - "configure burnout detection for users"
  - "build task management with anomaly detection"
  - "setup JWT authentication for enterprise app"
  - "implement AI-powered ticket classification"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript/Node.js application that provides centralized user, task, and ticket management with integrated AI capabilities. The system uses FastAPI + scikit-learn for ML features including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

**Key capabilities:**
- User authentication with JWT
- Role-based access control (Admin/User)
- Kanban task management with time tracking
- AI-powered ticket classification and routing
- Real-time risk and anomaly detection
- Burnout prediction using workload analysis
- MongoDB for data persistence

## Installation

### Prerequisites

```bash
# Ensure you have installed:
node --version  # v14 or higher
npm --version
python --version  # 3.8 or higher
mongod --version
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

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file for ML service
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
EOF

# Start frontend
npm start
# Frontend runs at http://localhost:3000
```

## Architecture

The system follows a three-tier architecture:

1. **Frontend (React)**: User interface for dashboards, task management
2. **Backend (Node.js/Express)**: REST API, authentication, business logic
3. **ML Service (FastAPI/Python)**: AI models for analytics and predictions

## Key API Endpoints

### Authentication

```javascript
// Login user
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

// Register new user
const register = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(userData)
  });
  return await response.json();
};
```

### User Management (Admin)

```javascript
// Get all users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return await response.json();
};

// Create user
const createUser = async (userData, token) => {
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

// Update user role
const updateUserRole = async (userId, role, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}/role`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ role })
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
// Get user tasks
const getUserTasks = async (token) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

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
      assignedTo: taskData.assignedTo,
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'in_progress', 'done'
    })
  });
  return await response.json();
};

// Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status })
  });
  return await response.json();
};

// Track time on task
const trackTaskTime = async (taskId, timeSpent, token) => {
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
      priority: ticketData.priority
    })
  });
  return await response.json();
};

// Get user tickets
const getUserTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Update ticket status (Admin)
const updateTicketStatus = async (ticketId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'open', 'in_progress', 'resolved', 'closed'
  });
  return await response.json();
};
```

## AI/ML Features

### Risk Detection

```javascript
// Predict user risk level
const predictUserRisk = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId,
      features: {
        taskCompletionRate: 0.75,
        averageTaskDelay: 2.5,
        ticketCount: 5,
        loginFrequency: 20,
        workloadScore: 8.5
      }
    })
  });
  const result = await response.json();
  // result: { riskLevel: 'low' | 'medium' | 'high', probability: 0.85 }
  return result;
};
```

### Anomaly Detection

```javascript
// Detect anomalous behavior
const detectAnomaly = async (userActivity, token) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId: userActivity.userId,
      loginTime: userActivity.loginTime,
      loginLocation: userActivity.loginLocation,
      activityPattern: userActivity.activityPattern
    })
  });
  const result = await response.json();
  // result: { isAnomaly: true, anomalyScore: 0.92, reason: 'Unusual login time' }
  return result;
};
```

### Burnout Detection

```javascript
// Analyze burnout risk
const analyzeBurnout = async (userId, token) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-analysis', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId,
      metrics: {
        overtimeHours: 45,
        taskLoad: 25,
        missedDeadlines: 3,
        stressIndicators: 7
      }
    })
  });
  const result = await response.json();
  // result: { burnoutRisk: 'high', score: 0.78, recommendations: [...] }
  return result;
};
```

### AI Ticket Classification

```javascript
// Auto-classify and route ticket
const classifyTicket = async (ticketContent, token) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketContent.subject,
      description: ticketContent.description
    })
  });
  const result = await response.json();
  // result: { category: 'technical', priority: 'high', assignTo: 'team-A' }
  return result;
};
```

### Predictive Project Insights

```javascript
// Predict project delays
const predictProjectDelay = async (projectData, token) => {
  const response = await fetch('http://localhost:8000/api/ml/project-insights', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      projectId: projectData.projectId,
      completionRate: projectData.completionRate,
      remainingTasks: projectData.remainingTasks,
      daysUntilDeadline: projectData.daysUntilDeadline,
      teamVelocity: projectData.teamVelocity
    })
  });
  const result = await response.json();
  // result: { delayProbability: 0.65, estimatedDelay: 5, recommendations: [...] }
  return result;
};
```

## Common Patterns

### Protected Route Component (React)

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### API Service Helper

```javascript
// services/api.js
class ApiService {
  constructor(baseURL) {
    this.baseURL = baseURL;
  }

  async request(endpoint, options = {}) {
    const token = localStorage.getItem('token');
    const headers = {
      'Content-Type': 'application/json',
      ...options.headers
    };

    if (token) {
      headers['Authorization'] = `Bearer ${token}`;
    }

    const response = await fetch(`${this.baseURL}${endpoint}`, {
      ...options,
      headers
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message || 'Request failed');
    }

    return await response.json();
  }

  get(endpoint) {
    return this.request(endpoint);
  }

  post(endpoint, data) {
    return this.request(endpoint, {
      method: 'POST',
      body: JSON.stringify(data)
    });
  }

  patch(endpoint, data) {
    return this.request(endpoint, {
      method: 'PATCH',
      body: JSON.stringify(data)
    });
  }

  delete(endpoint) {
    return this.request(endpoint, { method: 'DELETE' });
  }
}

export const api = new ApiService(process.env.REACT_APP_API_URL);
export const mlApi = new ApiService(process.env.REACT_APP_ML_URL);
```

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';
import { api } from '../services/api';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    try {
      const data = await api.get('/tasks');
      const grouped = {
        todo: data.filter(t => t.status === 'todo'),
        in_progress: data.filter(t => t.status === 'in_progress'),
        done: data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to load tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await api.patch(`/tasks/${taskId}/status`, { status: newStatus });
      loadTasks();
    } catch (error) {
      console.error('Failed to move task:', error);
    }
  };

  const renderColumn = (status, title) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      {tasks[status].map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <select 
            value={status} 
            onChange={(e) => moveTask(task._id, e.target.value)}
          >
            <option value="todo">To Do</option>
            <option value="in_progress">In Progress</option>
            <option value="done">Done</option>
          </select>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do')}
      {renderColumn('in_progress', 'In Progress')}
      {renderColumn('done', 'Done')}
    </div>
  );
};

export default KanbanBoard;
```

### Task Timer Component

```javascript
import React, { useState, useEffect } from 'react';
import { api } from '../services/api';

const TaskTimer = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [time, setTime] = useState(0);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setTime(t => t + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const startTimer = () => setIsRunning(true);
  
  const stopTimer = async () => {
    setIsRunning(false);
    const minutes = Math.floor(time / 60);
    if (minutes > 0) {
      try {
        await api.post(`/tasks/${taskId}/time`, { timeSpent: minutes });
        console.log(`Logged ${minutes} minutes`);
      } catch (error) {
        console.error('Failed to log time:', error);
      }
    }
    setTime(0);
  };

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="task-timer">
      <div className="timer-display">{formatTime(time)}</div>
      <button onClick={isRunning ? stopTimer : startTimer}>
        {isRunning ? 'Stop & Save' : 'Start Timer'}
      </button>
    </div>
  );
};

export default TaskTimer;
```

### Admin Analytics Dashboard

```javascript
import React, { useState, useEffect } from 'react';
import { api, mlApi } from '../services/api';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeTasks: 0,
    pendingTickets: 0,
    riskUsers: []
  });

  useEffect(() => {
    loadAnalytics();
  }, []);

  const loadAnalytics = async () => {
    try {
      const users = await api.get('/users');
      const tasks = await api.get('/tasks');
      const tickets = await api.get('/tickets');

      // Get risk predictions for all users
      const riskPromises = users.map(user => 
        mlApi.post('/api/ml/risk-prediction', {
          userId: user._id,
          features: calculateUserFeatures(user, tasks)
        }).catch(() => null)
      );
      const riskResults = await Promise.all(riskPromises);
      
      const highRiskUsers = users.filter((user, idx) => 
        riskResults[idx]?.riskLevel === 'high'
      );

      setAnalytics({
        totalUsers: users.length,
        activeTasks: tasks.filter(t => t.status !== 'done').length,
        pendingTickets: tickets.filter(t => t.status === 'open').length,
        riskUsers: highRiskUsers
      });
    } catch (error) {
      console.error('Failed to load analytics:', error);
    }
  };

  const calculateUserFeatures = (user, tasks) => {
    const userTasks = tasks.filter(t => t.assignedTo === user._id);
    const completedTasks = userTasks.filter(t => t.status === 'done');
    
    return {
      taskCompletionRate: userTasks.length > 0 
        ? completedTasks.length / userTasks.length 
        : 0,
      workloadScore: userTasks.length,
      averageTaskDelay: calculateAverageDelay(userTasks)
    };
  };

  const calculateAverageDelay = (tasks) => {
    const delayedTasks = tasks.filter(t => 
      new Date(t.dueDate) < new Date() && t.status !== 'done'
    );
    return delayedTasks.length / (tasks.length || 1);
  };

  return (
    <div className="admin-dashboard">
      <div className="stats-grid">
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
        <div className="stat-card alert">
          <h3>High Risk Users</h3>
          <p>{analytics.riskUsers.length}</p>
        </div>
      </div>
      
      {analytics.riskUsers.length > 0 && (
        <div className="risk-alerts">
          <h3>⚠️ Users Requiring Attention</h3>
          <ul>
            {analytics.riskUsers.map(user => (
              <li key={user._id}>
                {user.name} - {user.email}
              </li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AdminDashboard;
```

## Configuration

### Backend Configuration (backend/.env)

```bash
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management

# Authentication
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=${SMTP_HOST}
SMTP_PORT=${SMTP_PORT}
SMTP_USER=${SMTP_USER}
SMTP_PASS=${SMTP_PASS}

# Logging
LOG_LEVEL=info
```

### ML Service Configuration (ml-service/.env)

```bash
# Model configuration
MODEL_PATH=./models
RETRAIN_INTERVAL=86400  # 24 hours in seconds

# API
API_HOST=0.0.0.0
API_PORT=8000

# Logging
LOG_LEVEL=INFO

# Feature thresholds
ANOMALY_THRESHOLD=0.85
RISK_HIGH_THRESHOLD=0.75
BURNOUT_THRESHOLD=0.70
```

### Frontend Configuration (frontend/.env)

```bash
# API endpoints
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000

# App settings
REACT_APP_NAME=Enterprise User Management
REACT_APP_VERSION=1.0.0
```

## Database Schema Examples

### User Schema

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  position: String,
  isActive: { type: Boolean, default: true },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Schema

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  status: { 
    type: String, 
    enum: ['todo', 'in_progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  assignedTo: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'User', 
    required: true 
  },
  createdBy: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'User' 
  },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in minutes
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Schema

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  status: { 
    type: String, 
    enum: ['open', 'in_progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'urgent'], 
    default: 'medium' 
  },
  category: String, // Auto-filled by AI
  createdBy: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'User', 
    required: true 
  },
  assignedTo: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'User' 
  },
  comments: [{
    user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
    text: String,
    createdAt: { type: Date, default: Date.now }
  }],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Troubleshooting

### JWT Token Expired

```javascript
// Add token refresh logic
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
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    // Retry connection after 5 seconds
    setTimeout(connectDB, 5000);
  }
};

module.exports = connectDB;
```

### ML Service Not Responding

```python
# ml-service/main.py - Add health check
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

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
    return {"status": "healthy", "service": "ml-service"}

# Check with: curl http://localhost:8000/health
```

### CORS Issues

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');

const app = express();

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));

app.use(express.json());
```

### High Memory Usage in ML Service

```python
# ml-service/main.py - Implement model caching
from functools import lru_cache
import joblib

@lru_cache(maxsize=1)
def load_model(model_name):
    return joblib.load(f'./models/{model_name}.pkl')

# Use cached model
def predict_risk(features):
    model = load_model('risk_predictor')
    return model.predict([features])
```

### Task Status Not Updating

```javascript
// Add optimistic UI updates
const updateTaskStatus = async (taskId, newStatus) => {
  // Optimistically update UI
  setTasks(prevTasks => {
    const updated = { ...prevTasks };
    Object.keys(updated).forEach(status => {
      updated[status] = updated[status].map(task => 
        task._id === taskId ? { ...task, status: newStatus } : task
      );
    });
    return updated;
  });

  try {
    await api.patch(`/tasks/${taskId}/status`, { status: newStatus });
  } catch (error) {
    // Revert on error
    console.error('Failed to update task:', error);
    loadTasks(); // Reload from server
  }
};
```

### Audit Logging

```javascript
// backend/middleware/auditLog.js
const AuditLog = require('../models/AuditLog');

const auditLog = (action) => async (req, res, next) => {
  res.on('finish', async () => {
    
