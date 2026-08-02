---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, and risk prediction
triggers:
  - "build a user management system with AI analytics"
  - "implement enterprise task tracking with burnout detection"
  - "create admin dashboard with AI-powered insights"
  - "set up JWT authentication for user management"
  - "integrate AI ticket classification system"
  - "add anomaly detection to user management app"
  - "build Kanban board with time tracking"
  - "implement role-based access control system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management application with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights. Built with React, Node.js, FastAPI, and MongoDB.

## What This Project Does

This system provides:
- **User Management**: Secure authentication, role-based access control, user CRUD operations
- **Task Management**: Kanban boards, time tracking, task assignment and monitoring
- **Support System**: Ticket creation, AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, security alerts

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

### Backend Setup (Node.js)

```bash
cd backend
npm install
```

Create `.env` file:
```bash
PORT=5000
MONGODB_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=24h
NODE_ENV=development
```

Start backend:
```bash
npm start
# Runs at http://localhost:5000
```

### ML Service Setup (FastAPI)

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:
```bash
ML_SERVICE_PORT=8000
MODEL_PATH=./models
BACKEND_URL=http://localhost:5000
```

Start ML service:
```bash
uvicorn main:app --reload
# Runs at http://localhost:8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install
```

Create `.env` file:
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
# Runs at http://localhost:3000
```

## Key API Endpoints

### Authentication APIs (Backend)

**Register User**
```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/register`, {
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
```

**Login**
```javascript
// POST /api/auth/login
const loginUser = async (credentials) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
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

### User Management APIs

**Get All Users (Admin)**
```javascript
// GET /api/users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

**Update User**
```javascript
// PUT /api/users/:id
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

**Delete User (Admin)**
```javascript
// DELETE /api/users/:id
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management APIs

**Create Task**
```javascript
// POST /api/tasks
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
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
      status: 'todo', // 'todo', 'inprogress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};
```

**Update Task Status**
```javascript
// PATCH /api/tasks/:id/status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
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

**Track Time on Task**
```javascript
// POST /api/tasks/:id/time
const trackTime = async (taskId, timeSpent) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ 
      timeSpent: timeSpent, // in minutes
      timestamp: new Date().toISOString()
    })
  });
  return response.json();
};
```

### Support Ticket APIs

**Create Ticket**
```javascript
// POST /api/tickets
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
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
```

**Get User Tickets**
```javascript
// GET /api/tickets/user/:userId
const getUserTickets = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tickets/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### AI/ML Service APIs

**Classify Ticket (AI)**
```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/classify-ticket`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: ticketText.substring(0, 100)
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.85 }
  return data;
};
```

**Predict User Risk**
```javascript
// POST /api/ml/predict-risk
const predictUserRisk = async (userData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/predict-risk`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.id,
      loginFrequency: userData.loginFrequency,
      taskCompletionRate: userData.taskCompletionRate,
      averageResponseTime: userData.avgResponseTime,
      failedLoginAttempts: userData.failedLogins
    })
  });
  const data = await response.json();
  // Returns: { riskScore: 0.75, riskLevel: 'high', factors: [...] }
  return data;
};
```

**Detect Burnout**
```javascript
// POST /api/ml/detect-burnout
const detectBurnout = async (userId) => {
  const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/detect-burnout`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      tasksInProgress: 15,
      avgHoursPerDay: 10,
      missedDeadlines: 3,
      weeksSinceLastBreak: 6
    })
  });
  const data = await response.json();
  // Returns: { burnoutScore: 0.82, recommendation: 'immediate_action' }
  return data;
};
```

**Predict Project Delay**
```javascript
// POST /api/ml/predict-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/predict-delay`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.id,
      tasksCompleted: projectData.completed,
      tasksRemaining: projectData.remaining,
      daysUntilDeadline: projectData.daysLeft,
      teamVelocity: projectData.velocity
    })
  });
  const data = await response.json();
  // Returns: { delayProbability: 0.65, estimatedDelay: 5 }
  return data;
};
```

**Detect Anomalies**
```javascript
// POST /api/ml/detect-anomaly
const detectAnomaly = async (userActivity) => {
  const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/detect-anomaly`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userActivity.userId,
      loginTime: userActivity.timestamp,
      ipAddress: userActivity.ip,
      location: userActivity.location,
      deviceType: userActivity.device,
      activityPattern: userActivity.pattern
    })
  });
  const data = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.9, type: 'unusual_location' }
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
  const userRole = localStorage.getItem('role');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && userRole !== requiredRole) {
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

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`, {
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
    await updateTaskStatus(taskId, newStatus);
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      <div className="column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <TaskCard key={task.id} task={task} onMove={moveTask} />
        ))}
      </div>
      <div className="column">
        <h3>In Progress</h3>
        {tasks.inprogress.map(task => (
          <TaskCard key={task.id} task={task} onMove={moveTask} />
        ))}
      </div>
      <div className="column">
        <h3>Done</h3>
        {tasks.done.map(task => (
          <TaskCard key={task.id} task={task} onMove={moveTask} />
        ))}
      </div>
    </div>
  );
};
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isActive, setIsActive] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isActive) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else if (!isActive && seconds !== 0) {
      clearInterval(interval);
    }
    return () => clearInterval(interval);
  }, [isActive, seconds]);

  const toggle = () => setIsActive(!isActive);

  const reset = async () => {
    const minutes = Math.floor(seconds / 60);
    if (minutes > 0) {
      await trackTime(taskId, minutes);
    }
    setSeconds(0);
    setIsActive(false);
  };

  const formatTime = (s) => {
    const hrs = Math.floor(s / 3600);
    const mins = Math.floor((s % 3600) / 60);
    const secs = s % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer">{formatTime(seconds)}</div>
      <button onClick={toggle}>{isActive ? 'Pause' : 'Start'}</button>
      <button onClick={reset}>Save & Reset</button>
    </div>
  );
};
```

### AI-Powered Ticket Creation

```javascript
const SmartTicketForm = () => {
  const [formData, setFormData] = useState({
    subject: '',
    description: '',
    priority: '',
    category: ''
  });
  const [aiSuggestions, setAiSuggestions] = useState(null);

  const handleDescriptionChange = async (e) => {
    const text = e.target.value;
    setFormData({ ...formData, description: text });

    // Auto-classify when description is long enough
    if (text.length > 50) {
      const classification = await classifyTicket(text);
      setAiSuggestions(classification);
      setFormData({
        ...formData,
        description: text,
        category: classification.category,
        priority: classification.priority
      });
    }
  };

  const submitTicket = async (e) => {
    e.preventDefault();
    const result = await createTicket(formData);
    if (result.success) {
      alert('Ticket created successfully!');
      setFormData({ subject: '', description: '', priority: '', category: '' });
    }
  };

  return (
    <form onSubmit={submitTicket}>
      <input
        type="text"
        placeholder="Subject"
        value={formData.subject}
        onChange={(e) => setFormData({ ...formData, subject: e.target.value })}
        required
      />
      <textarea
        placeholder="Describe your issue..."
        value={formData.description}
        onChange={handleDescriptionChange}
        required
      />
      {aiSuggestions && (
        <div className="ai-suggestions">
          <p>AI Suggestions:</p>
          <p>Category: {aiSuggestions.category}</p>
          <p>Priority: {aiSuggestions.priority}</p>
          <p>Confidence: {(aiSuggestions.confidence * 100).toFixed(0)}%</p>
        </div>
      )}
      <select
        value={formData.priority}
        onChange={(e) => setFormData({ ...formData, priority: e.target.value })}
        required
      >
        <option value="">Select Priority</option>
        <option value="low">Low</option>
        <option value="medium">Medium</option>
        <option value="high">High</option>
      </select>
      <button type="submit">Create Ticket</button>
    </form>
  );
};
```

### Admin Analytics Dashboard

```javascript
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeTasks: 0,
    openTickets: 0,
    riskUsers: []
  });

  useEffect(() => {
    loadAnalytics();
  }, []);

  const loadAnalytics = async () => {
    const token = localStorage.getItem('token');
    
    // Fetch all users
    const usersRes = await fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const users = await usersRes.json();
    
    // Fetch tasks
    const tasksRes = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const tasks = await tasksRes.json();
    
    // Fetch tickets
    const ticketsRes = await fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const tickets = await ticketsRes.json();
    
    // Check for high-risk users
    const riskUsers = [];
    for (const user of users) {
      const riskAnalysis = await predictUserRisk({
        id: user.id,
        loginFrequency: user.loginFrequency || 0,
        taskCompletionRate: user.taskCompletionRate || 0,
        avgResponseTime: user.avgResponseTime || 0,
        failedLogins: user.failedLoginAttempts || 0
      });
      
      if (riskAnalysis.riskLevel === 'high') {
        riskUsers.push({ ...user, riskScore: riskAnalysis.riskScore });
      }
    }
    
    setAnalytics({
      totalUsers: users.length,
      activeTasks: tasks.filter(t => t.status !== 'done').length,
      openTickets: tickets.filter(t => t.status === 'open').length,
      riskUsers: riskUsers
    });
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
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
          <h3>Open Tickets</h3>
          <p>{analytics.openTickets}</p>
        </div>
      </div>
      
      {analytics.riskUsers.length > 0 && (
        <div className="risk-alerts">
          <h2>⚠️ High Risk Users</h2>
          {analytics.riskUsers.map(user => (
            <div key={user.id} className="risk-user-card">
              <p>{user.name} ({user.email})</p>
              <p>Risk Score: {(user.riskScore * 100).toFixed(0)}%</p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};
```

## Backend Implementation Examples

### JWT Authentication Middleware (Node.js)

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return res.status(401).json({ message: 'No token provided' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### User Model (MongoDB/Mongoose)

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  loginFrequency: { type: Number, default: 0 },
  taskCompletionRate: { type: Number, default: 0 },
  avgResponseTime: { type: Number, default: 0 },
  failedLoginAttempts: { type: Number, default: 0 },
  lastLogin: { type: Date },
  createdAt: { type: Date, default: Date.now }
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
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
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
  dueDate: { type: Date },
  timeTracked: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  category: { 
    type: String, 
    enum: ['technical', 'billing', 'general', 'urgent'],
    default: 'general'
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  status: { 
    type: String, 
    enum: ['open', 'inprogress', 'resolved', 'closed'],
    default: 'open'
  },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: {
    confidence: Number,
    suggestedCategory: String,
    suggestedPriority: String
  },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: { type: Date }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### Auth Routes

```javascript
// routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const User = require('../models/User');
const router = express.Router();

// Register
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const user = new User({ name, email, password, role });
    await user.save();
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ 
      token, 
      user: { id: user._id, name, email, role } 
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const isMatch = await user.comparePassword(password);
    if (!isMatch) {
      user.failedLoginAttempts += 1;
      await user.save();
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    user.lastLogin = new Date();
    user.loginFrequency += 1;
    user.failedLoginAttempts = 0;
    await user.save();
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ 
      token, 
      user: { id: user._id, name: user.name, email, role: user.role } 
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation (Python/FastAPI)

### Main FastAPI Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly
import joblib
import os

app = FastAPI()

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Ticket classification model
class TicketClassificationRequest(BaseModel):
    text: str
    subject: str
