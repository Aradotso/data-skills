---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "create task tracking with burnout detection"
  - "implement ticket classification system"
  - "build user dashboard with kanban board"
  - "add anomaly detection to user system"
  - "configure JWT authentication for enterprise app"
  - "deploy user management with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user/task management with AI-powered insights. The system provides:

- **User Management**: Role-based access control, authentication, user CRUD operations
- **Task Tracking**: Kanban boards, time tracking, assignment workflows
- **Support Tickets**: AI-based classification, routing, and priority management
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, predictive project insights
- **Admin Dashboard**: Organization-wide analytics, audit logs, security alerts

The architecture consists of three services:
1. **Frontend**: React.js application (port 3000)
2. **Backend**: Node.js REST API with MongoDB (port 5000)
3. **ML Service**: FastAPI with scikit-learn and River (port 8000)

## Installation

### Prerequisites

```bash
# Required
node --version  # v14+ required
python --version  # Python 3.8+ required
mongod --version  # MongoDB 4.4+ required
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install all services
npm run install-all  # If root package.json exists

# Or install individually
cd backend && npm install
cd ../frontend && npm install
cd ../ml-service && pip install -r requirements.txt
```

### Environment Configuration

**Backend** (`backend/.env`):
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend** (`frontend/.env`):
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

**ML Service** (`ml-service/.env`):
```env
MODEL_PATH=./models
MONGO_URI=mongodb://localhost:27017/enterprise_user_mgmt
LOG_LEVEL=info
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

### Production Build

```bash
# Frontend
cd frontend
npm run build

# Backend (with PM2)
cd backend
npm install -g pm2
pm2 start server.js --name enterprise-backend

# ML Service (with uvicorn)
cd ml-service
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

## Backend API Usage

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
      role: userData.role || 'user' // 'user' or 'admin'
    })
  });
  return response.json();
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

// Protected route example
const getProfile = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users/profile', {
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};
```

### User Management (Admin)

```javascript
// Get all users (admin only)
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/admin/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/admin/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/admin/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management

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
      status: 'todo', // 'todo', 'inprogress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// Update task status
const updateTaskStatus = async (taskId, newStatus, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};

// Track time on task
const trackTime = async (taskId, timeData, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      startTime: timeData.startTime,
      endTime: timeData.endTime,
      duration: timeData.duration // in minutes
    })
  });
  return response.json();
};
```

### Ticket Management

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
      category: ticketData.category // 'technical', 'billing', 'general'
    })
  });
  return response.json();
};

// Get user tickets
const getUserTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets/user', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Update ticket status (admin)
const updateTicketStatus = async (ticketId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ 
      status // 'open', 'in-progress', 'resolved', 'closed'
    })
  });
  return response.json();
};
```

## ML Service API Usage

### AI-Powered Ticket Classification

```javascript
// Classify ticket using AI
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: "Login issues"
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.92 }
  return data;
};
```

### Risk Prediction

```javascript
// Predict user risk based on behavior
const predictUserRisk = async (userId, behaviorData) => {
  const response = await fetch('http://localhost:8000/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      login_frequency: behaviorData.loginFrequency,
      failed_logins: behaviorData.failedLogins,
      unusual_hours: behaviorData.unusualHours,
      data_access_volume: behaviorData.dataAccessVolume
    })
  });
  const data = await response.json();
  // Returns: { risk_level: 'high', risk_score: 0.85, factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// Detect anomalies in user activity
const detectAnomalies = async (activityData) => {
  const response = await fetch('http://localhost:8000/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: activityData.userId,
      timestamp: activityData.timestamp,
      action_type: activityData.actionType,
      ip_address: activityData.ipAddress,
      location: activityData.location
    })
  });
  const data = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.78, reason: "Unusual location" }
  return data;
};
```

### Burnout Detection

```javascript
// Analyze user workload for burnout risk
const detectBurnout = async (userId, workloadData) => {
  const response = await fetch('http://localhost:8000/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      tasks_count: workloadData.tasksCount,
      avg_task_duration: workloadData.avgTaskDuration,
      overtime_hours: workloadData.overtimeHours,
      days_since_break: workloadData.daysSinceBreak,
      missed_deadlines: workloadData.missedDeadlines
    })
  });
  const data = await response.json();
  // Returns: { burnout_risk: 'medium', score: 0.65, recommendations: [...] }
  return data;
};
```

### Predictive Project Insights

```javascript
// Predict project delay probability
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.projectId,
      completion_percentage: projectData.completionPercentage,
      days_remaining: projectData.daysRemaining,
      team_size: projectData.teamSize,
      task_velocity: projectData.taskVelocity,
      blocker_count: projectData.blockerCount
    })
  });
  const data = await response.json();
  // Returns: { delay_probability: 0.72, estimated_delay_days: 5, suggestions: [...] }
  return data;
};
```

## React Frontend Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
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
      const response = await fetch(`${process.env.REACT_APP_API_URL}/users/profile`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data);
    } catch (error) {
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
    setToken(data.token);
    localStorage.setItem('token', data.token);
    setUser(data.user);
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
// src/components/KanbanBoard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const KanbanBoard = () => {
  const { token } = useContext(AuthContext);
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });

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
      inprogress: data.filter(t => t.status === 'inprogress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks(); // Refresh
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

### AI-Powered Ticket Form

```javascript
// src/components/TicketForm.js
import React, { useState, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const TicketForm = () => {
  const { token } = useContext(AuthContext);
  const [formData, setFormData] = useState({ subject: '', description: '' });
  const [aiSuggestion, setAiSuggestion] = useState(null);

  const classifyTicket = async () => {
    const response = await fetch(`${process.env.REACT_APP_ML_URL}/classify-ticket`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        text: formData.description,
        subject: formData.subject
      })
    });
    const data = await response.json();
    setAiSuggestion(data);
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    const response = await fetch(`${process.env.REACT_APP_API_URL}/tickets`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        ...formData,
        category: aiSuggestion?.category || 'general',
        priority: aiSuggestion?.priority || 'medium'
      })
    });
    
    if (response.ok) {
      alert('Ticket created successfully!');
      setFormData({ subject: '', description: '' });
      setAiSuggestion(null);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        placeholder="Subject"
        value={formData.subject}
        onChange={(e) => setFormData({ ...formData, subject: e.target.value })}
      />
      <textarea
        placeholder="Description"
        value={formData.description}
        onChange={(e) => setFormData({ ...formData, description: e.target.value })}
        onBlur={classifyTicket}
      />
      
      {aiSuggestion && (
        <div className="ai-suggestion">
          <p>AI Suggestion: Category: {aiSuggestion.category}, 
             Priority: {aiSuggestion.priority}</p>
        </div>
      )}
      
      <button type="submit">Create Ticket</button>
    </form>
  );
};

export default TicketForm;
```

## Database Schema Examples

### User Model (MongoDB)

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
  isActive: { type: Boolean, default: true },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now },
  metadata: {
    loginCount: { type: Number, default: 0 },
    failedLoginAttempts: { type: Number, default: 0 },
    lastFailedLogin: Date
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  status: { 
    type: String, 
    enum: ['todo', 'inprogress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  dueDate: Date,
  completedAt: Date,
  timeTracking: [{
    startTime: Date,
    endTime: Date,
    duration: Number // in minutes
  }],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

taskSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  category: { 
    type: String, 
    enum: ['technical', 'billing', 'general', 'hr'], 
    default: 'general' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: {
    confidence: Number,
    suggestedCategory: String,
    suggestedPriority: String
  },
  comments: [{
    user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
    text: String,
    timestamp: { type: Date, default: Date.now }
  }],
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Integration Patterns

### Dashboard Analytics Fetcher

```javascript
// src/utils/dashboardAPI.js
export const fetchDashboardData = async (token) => {
  const headers = { 'Authorization': `Bearer ${token}` };
  
  // Fetch multiple endpoints in parallel
  const [users, tasks, tickets, analytics] = await Promise.all([
    fetch(`${process.env.REACT_APP_API_URL}/admin/users`, { headers }).then(r => r.json()),
    fetch(`${process.env.REACT_APP_API_URL}/tasks`, { headers }).then(r => r.json()),
    fetch(`${process.env.REACT_APP_API_URL}/tickets`, { headers }).then(r => r.json()),
    fetch(`${process.env.REACT_APP_ML_URL}/analytics/summary`, { 
      method: 'POST',
      headers: { 'Content-Type': 'application/json' }
    }).then(r => r.json())
  ]);
  
  return {
    totalUsers: users.length,
    activeUsers: users.filter(u => u.isActive).length,
    totalTasks: tasks.length,
    completedTasks: tasks.filter(t => t.status === 'done').length,
    openTickets: tickets.filter(t => t.status === 'open').length,
    aiInsights: analytics
  };
};
```

### Real-time Anomaly Monitor

```javascript
// src/hooks/useAnomalyMonitor.js
import { useEffect, useState } from 'react';

export const useAnomalyMonitor = (userId, token) => {
  const [anomalies, setAnomalies] = useState([]);

  useEffect(() => {
    const checkAnomalies = async () => {
      // Get recent user activity
      const response = await fetch(
        `${process.env.REACT_APP_API_URL}/users/${userId}/activity`,
        { headers: { 'Authorization': `Bearer ${token}` } }
      );
      const activities = await response.json();
      
      // Check each activity for anomalies
      for (const activity of activities) {
        const anomalyCheck = await fetch(
          `${process.env.REACT_APP_ML_URL}/detect-anomaly`,
          {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              user_id: userId,
              timestamp: activity.timestamp,
              action_type: activity.actionType,
              ip_address: activity.ipAddress,
              location: activity.location
            })
          }
        );
        const result = await anomalyCheck.json();
        
        if (result.is_anomaly) {
          setAnomalies(prev => [...prev, { activity, result }]);
        }
      }
    };

    // Check every 5 minutes
    const interval = setInterval(checkAnomalies, 5 * 60 * 1000);
    checkAnomalies(); // Initial check
    
    return () => clearInterval(interval);
  }, [userId, token]);

  return anomalies;
};
```

### Burnout Alert System

```javascript
// src/components/BurnoutAlert.js
import React, { useEffect, useState, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const BurnoutAlert = () => {
  const { user, token } = useContext(AuthContext);
  const [burnoutData, setBurnoutData] = useState(null);

  useEffect(() => {
    if (!user) return;

    const checkBurnout = async () => {
      // Get user workload data
      const tasksResponse = await fetch(
        `${process.env.REACT_APP_API_URL}/tasks/user/${user._id}/stats`,
        { headers: { 'Authorization': `Bearer ${token}` } }
      );
      const workloadData = await tasksResponse.json();

      // Check for burnout
      const burnoutResponse = await fetch(
        `${process.env.REACT_APP_ML_URL}/detect-burnout`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            user_id: user._id,
            tasks_count: workloadData.activeTasksCount,
            avg_task_duration: workloadData.avgDuration,
            overtime_hours: workloadData.overtimeHours,
            days_since_break: workloadData.daysSinceBreak,
            missed_deadlines: workloadData.missedDeadlines
          })
        }
      );
      const result = await burnoutResponse.json();
      setBurnoutData(result);
    };

    checkBurnout();
    const interval = setInterval(checkBurnout, 24 * 60 * 60 * 1000); // Daily
    
    return () => clearInterval(interval);
  }, [user, token]);

  if (!burnoutData || burnoutData.burnout_risk === 'low') return null;

  return (
    <div className={`burnout-alert ${burnoutData.burnout_risk}`}>
      <h4>⚠️ Burnout Risk: {burnoutData.burnout_risk.toUpperCase()}</h4>
      <p>Score: {(burnoutData.score * 100).toFixed(0)}%</p>
      <ul>
        {burnoutData.recommendations?.map((rec, idx) => (
          <li key={idx}>{rec}</li>
        ))}
      </ul>
    </div>
  );
};

export default BurnoutAlert;
```

## Troubleshooting

### JWT Token Expiration

```javascript
// src/utils/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

// Add token to all requests
api.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle 401 errors (token expired)
api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
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
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    // Retry after 5 seconds
    setTimeout(connectDB, 5000);
  }
};

// Handle connection events
mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected. Attempting to reconnect...');
  connectDB();
});

module.exports = connectDB;
```

### ML Service Model Loading

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
import joblib
import os
from pathlib import Path

app = FastAPI()

# Load models at startup
models = {}

@app.on_event("startup")
async def load_models():
    model_path = Path(os
