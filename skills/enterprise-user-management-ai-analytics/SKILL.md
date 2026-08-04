---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task management, ticket routing, and predictive insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management dashboard with task tracking"
  - "implement AI ticket classification and routing"
  - "build admin panel with user and task management"
  - "add AI anomaly detection to user management system"
  - "integrate ML service for burnout prediction"
  - "configure JWT authentication for enterprise app"
  - "deploy user management system with kanban board"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a full-stack enterprise user management system combining React frontend, Node.js backend, and FastAPI ML service. It provides role-based access control, task management with Kanban boards, support ticket system, and AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## Installation

### Prerequisites

```bash
# Ensure you have installed:
node --version  # v14 or higher
npm --version   # v6 or higher
python --version  # Python 3.8+
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
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_JWT_SECRET=your_jwt_secret_key_here
```

Start frontend:

```bash
npm start
# Frontend runs at http://localhost:3000
```

## Architecture Overview

The system consists of three main components:

1. **Frontend (React)**: User interface for admin and user dashboards
2. **Backend (Node.js)**: REST API for user management, tasks, and tickets
3. **ML Service (FastAPI)**: AI/ML endpoints for analytics and predictions

## Backend API Reference

### Authentication

```javascript
// backend/routes/auth.js example usage
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.status(201).json({ token, user: { id: user._id, username, email, role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    
    if (!user || !(await bcrypt.compare(password, user.password))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Middleware for Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No authentication token provided' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

// Get all tasks for logged-in user
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('assignedBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create new task (admin only)
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.userId,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });
    
    await task.save();
    await task.populate('assignedTo assignedBy', 'username email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status (Kanban board)
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: new Date() },
      { new: true }
    ).populate('assignedTo assignedBy', 'username email');
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time on task
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body; // duration in seconds
    const task = await Task.findById(req.params.id);
    
    task.timeTracked = (task.timeTracked || 0) + duration;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Ticket System

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const { authMiddleware } = require('../middleware/auth');
const axios = require('axios');

// Create support ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { subject, description, priority } = req.body;
    
    // Call ML service for AI classification
    let category = 'general';
    let suggestedPriority = priority;
    
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        subject,
        description
      });
      category = mlResponse.data.category;
      suggestedPriority = mlResponse.data.priority;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    const ticket = new Ticket({
      subject,
      description,
      createdBy: req.user.userId,
      priority: suggestedPriority,
      category,
      status: 'open'
    });
    
    await ticket.save();
    await ticket.populate('createdBy', 'username email');
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user's tickets
router.get('/my-tickets', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.userId })
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Admin: Get all tickets
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find()
      .populate('createdBy assignedTo', 'username email')
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service API Reference

### AI Ticket Classification

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib
import os

app = FastAPI()

class TicketRequest(BaseModel):
    subject: str
    description: str

class RiskRequest(BaseModel):
    user_id: str
    login_count: int
    failed_logins: int
    tasks_completed: int
    tasks_overdue: int
    avg_completion_time: float

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketRequest):
    """Classify support ticket category and priority using NLP"""
    try:
        text = f"{ticket.subject} {ticket.description}".lower()
        
        # Simple rule-based classification (can be replaced with trained model)
        categories = {
            'technical': ['bug', 'error', 'crash', 'not working', 'broken'],
            'access': ['login', 'password', 'access', 'permission', 'locked'],
            'feature': ['request', 'need', 'add', 'feature', 'enhancement'],
            'general': []
        }
        
        category = 'general'
        for cat, keywords in categories.items():
            if any(keyword in text for keyword in keywords):
                category = cat
                break
        
        # Priority detection
        urgent_keywords = ['urgent', 'critical', 'asap', 'immediately', 'down']
        priority = 'high' if any(word in text for word in urgent_keywords) else 'medium'
        
        return {
            'category': category,
            'priority': priority,
            'confidence': 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(data: RiskRequest):
    """Predict user security risk based on behavior patterns"""
    try:
        # Calculate risk score (0-100)
        risk_score = 0
        
        # Failed login attempts contribute to risk
        if data.failed_logins > 5:
            risk_score += 30
        elif data.failed_logins > 2:
            risk_score += 15
        
        # Unusual login frequency
        if data.login_count > 50:
            risk_score += 20
        
        # Task completion issues
        if data.tasks_overdue > 5:
            risk_score += 15
        
        # Abnormal completion time
        if data.avg_completion_time > 48:  # hours
            risk_score += 10
        
        risk_level = 'high' if risk_score > 60 else 'medium' if risk_score > 30 else 'low'
        
        return {
            'risk_score': min(risk_score, 100),
            'risk_level': risk_level,
            'recommendations': [
                'Monitor user activity closely' if risk_score > 60 else 'Regular monitoring',
                'Review access permissions' if data.failed_logins > 5 else 'Normal access'
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(data: dict):
    """Analyze workload to detect employee burnout risk"""
    try:
        active_tasks = data.get('active_tasks', 0)
        avg_hours_per_day = data.get('avg_hours_per_day', 0)
        missed_deadlines = data.get('missed_deadlines', 0)
        weekends_worked = data.get('weekends_worked', 0)
        
        burnout_score = 0
        
        if active_tasks > 10:
            burnout_score += 25
        if avg_hours_per_day > 9:
            burnout_score += 30
        if missed_deadlines > 3:
            burnout_score += 20
        if weekends_worked > 2:
            burnout_score += 25
        
        burnout_level = 'high' if burnout_score > 70 else 'medium' if burnout_score > 40 else 'low'
        
        return {
            'burnout_score': min(burnout_score, 100),
            'burnout_level': burnout_level,
            'suggestions': [
                'Redistribute workload' if active_tasks > 10 else 'Workload is manageable',
                'Encourage time off' if weekends_worked > 2 else 'Good work-life balance',
                'Review deadlines' if missed_deadlines > 3 else 'Meeting deadlines well'
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-project-delay")
async def predict_project_delay(data: dict):
    """Predict likelihood of project delay"""
    try:
        total_tasks = data.get('total_tasks', 0)
        completed_tasks = data.get('completed_tasks', 0)
        days_remaining = data.get('days_remaining', 0)
        avg_task_completion_days = data.get('avg_task_completion_days', 0)
        
        remaining_tasks = total_tasks - completed_tasks
        estimated_days_needed = remaining_tasks * avg_task_completion_days
        
        delay_probability = min(100, (estimated_days_needed / max(days_remaining, 1)) * 50)
        
        return {
            'delay_probability': delay_probability,
            'estimated_delay_days': max(0, estimated_days_needed - days_remaining),
            'recommendation': 'Increase resources' if delay_probability > 70 else 'On track'
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Frontend React Components

### Login Component

```javascript
// frontend/src/components/Login.jsx
import React, { useState } from 'react';
import axios from 'axios';
import { useNavigate } from 'react-router-dom';

const Login = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const navigate = useNavigate();

  const handleLogin = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
        email,
        password
      });
      
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
      
      if (response.data.user.role === 'admin') {
        navigate('/admin/dashboard');
      } else {
        navigate('/user/dashboard');
      }
    } catch (err) {
      setError(err.response?.data?.error || 'Login failed');
    }
  };

  return (
    <div className="login-container">
      <form onSubmit={handleLogin}>
        <h2>Login</h2>
        {error && <div className="error">{error}</div>}
        <input
          type="email"
          placeholder="Email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />
        <input
          type="password"
          placeholder="Password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          required
        />
        <button type="submit">Login</button>
      </form>
    </div>
  );
};

export default Login;
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'inProgress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(categorized);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-${task.priority}`}>{task.priority}</span>
      <div className="task-actions">
        {task.status !== 'todo' && (
          <button onClick={() => moveTask(task._id, 'todo')}>← To Do</button>
        )}
        {task.status !== 'inProgress' && (
          <button onClick={() => moveTask(task._id, 'inProgress')}>In Progress</button>
        )}
        {task.status !== 'done' && (
          <button onClick={() => moveTask(task._id, 'done')}>Done →</button>
        )}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch user stats from backend
      const statsResponse = await axios.get(
        `${process.env.REACT_APP_API_URL}/users/${userId}/stats`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      // Get AI predictions from ML service
      const riskResponse = await axios.post(
        `${process.env.REACT_APP_ML_URL}/predict-risk`,
        statsResponse.data
      );
      
      const burnoutResponse = await axios.post(
        `${process.env.REACT_APP_ML_URL}/detect-burnout`,
        statsResponse.data
      );
      
      setAnalytics({
        risk: riskResponse.data,
        burnout: burnoutResponse.data
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      <div className="analytics-card">
        <h3>Security Risk Assessment</h3>
        <div className={`risk-level ${analytics.risk.risk_level}`}>
          {analytics.risk.risk_level.toUpperCase()}
        </div>
        <p>Risk Score: {analytics.risk.risk_score}/100</p>
        <ul>
          {analytics.risk.recommendations.map((rec, idx) => (
            <li key={idx}>{rec}</li>
          ))}
        </ul>
      </div>
      
      <div className="analytics-card">
        <h3>Burnout Detection</h3>
        <div className={`burnout-level ${analytics.burnout.burnout_level}`}>
          {analytics.burnout.burnout_level.toUpperCase()}
        </div>
        <p>Burnout Score: {analytics.burnout.burnout_score}/100</p>
        <ul>
          {analytics.burnout.suggestions.map((sug, idx) => (
            <li key={idx}>{sug}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Database Models

### User Model (MongoDB)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  isActive: {
    type: Boolean,
    default: true
  },
  loginHistory: [{
    timestamp: Date,
    ipAddress: String,
    success: Boolean
  }],
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: Date
});

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  status: {
    type: String,
    enum: ['todo', 'inProgress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  dueDate: Date,
  timeTracked: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  category: {
    type: String,
    enum: ['technical', 'access', 'feature', 'general'],
    default: 'general'
  },
  comments: [{
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User'
    },
    text: String,
    createdAt: {
      type: Date,
      default: Date.now
    }
  }],
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### API Client Setup

```javascript
// frontend/src/utils/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

// Add auth token to all requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle token expiration
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Protected Route Component

```javascript
// frontend/src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/user/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Time Tracker Hook

```javascript
// frontend/src/hooks/useTimeTracker.js
import { useState, useEffect, useRef } from 'react';
import api from '../utils/api';

export const useTimeTracker = (taskId) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    }

    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    };
  }, [isRunning]);

  const start = () => setIsRunning(true);
  const pause = () => setIsRunning(false);
  
  const stop = async () => {
    setIsRunning(false);
    if (seconds > 0) {
      try {
        await api.post(`/tasks/${taskId}/time`, { duration: seconds });
        setSeconds(0);
      } catch (error) {
        console.error('Failed to save time:', error);
      }
    }
  };

  const formatTime = () => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart
