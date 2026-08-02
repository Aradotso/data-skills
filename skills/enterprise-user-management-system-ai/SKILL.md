---
name: enterprise-user-management-system-ai
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management dashboard with AI features"
  - "implement task tracking with burnout detection"
  - "build support ticket system with AI classification"
  - "add AI-powered risk detection to user system"
  - "configure user management with kanban board"
  - "integrate ML analytics for project insights"
  - "deploy user management system with FastAPI ML"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform combining React frontend, Node.js backend, and FastAPI ML service. Provides role-based access control, task management with Kanban boards, support ticket system, and AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights.

## What It Does

- **User Management**: JWT-authenticated system with role-based access (Admin/User)
- **Task Tracking**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Real-time Insights**: Dashboard with performance metrics and alerts

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
python >= 3.8
mongodb >= 4.x
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
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

Create `ml-service/.env`:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
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
# Runs at http://localhost:3000
```

## Key API Endpoints

### Authentication (Backend)

```javascript
// POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@company.com",
  "password": "securepass123",
  "role": "user" // or "admin"
}

// POST /api/auth/login
{
  "email": "john@company.com",
  "password": "securepass123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management (Backend)

```javascript
// GET /api/users - List all users (Admin only)
// GET /api/users/:id - Get user by ID
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user (Admin only)
```

### Task Management (Backend)

```javascript
// GET /api/tasks - Get user's tasks
// POST /api/tasks - Create task
{
  "title": "Implement login feature",
  "description": "Add JWT authentication",
  "assignedTo": "user_id",
  "status": "todo", // todo, in_progress, done
  "priority": "high",
  "dueDate": "2026-05-01"
}

// PATCH /api/tasks/:id - Update task status
{
  "status": "in_progress",
  "timeSpent": 120 // minutes
}
```

### Support Tickets (Backend)

```javascript
// POST /api/tickets - Create ticket
{
  "title": "Unable to access dashboard",
  "description": "Getting 403 error",
  "priority": "high",
  "category": "technical"
}

// GET /api/tickets - Get tickets
// PATCH /api/tickets/:id - Update ticket
{
  "status": "in_progress",
  "assignedTo": "admin_id"
}
```

### AI Analytics (ML Service)

```python
# POST /api/ml/classify-ticket
{
  "title": "Password reset not working",
  "description": "Clicked forgot password but no email received"
}
# Returns: { "category": "technical", "priority": "medium", "confidence": 0.89 }

# POST /api/ml/detect-risk
{
  "userId": "user_id",
  "failedLogins": 5,
  "unusualActivity": true,
  "accessPatterns": ["night", "weekend"]
}
# Returns: { "riskScore": 0.76, "riskLevel": "high", "factors": [...] }

# POST /api/ml/detect-burnout
{
  "userId": "user_id",
  "tasksCompleted": 45,
  "hoursWorked": 65,
  "overtimeHours": 15,
  "missedDeadlines": 3
}
# Returns: { "burnoutScore": 0.82, "recommendation": "reduce_workload" }

# POST /api/ml/predict-delay
{
  "projectId": "proj_123",
  "tasksRemaining": 12,
  "averageCompletionTime": 4.5,
  "teamSize": 5,
  "complexityScore": 7
}
# Returns: { "delayProbability": 0.65, "estimatedDelay": 5 }
```

## Frontend Integration Examples

### Authentication Flow

```javascript
// src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const login = async (email, password) => {
  const response = await axios.post(`${API_URL}/api/auth/login`, {
    email,
    password
  });
  
  if (response.data.token) {
    localStorage.setItem('token', response.data.token);
    localStorage.setItem('user', JSON.stringify(response.data.user));
  }
  
  return response.data;
};

export const logout = () => {
  localStorage.removeItem('token');
  localStorage.removeItem('user');
};

export const getAuthHeader = () => {
  const token = localStorage.getItem('token');
  return token ? { Authorization: `Bearer ${token}` } : {};
};
```

### Task Management Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    in_progress: [],
    done: []
  });

  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks`, {
        headers: getAuthHeader()
      });
      
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, {});
      
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}`,
        { status: newStatus },
        { headers: getAuthHeader() }
      );
      
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {['todo', 'in_progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          {tasks[status]?.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status}
                onChange={(e) => updateTaskStatus(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in_progress">In Progress</option>
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

### AI-Powered Ticket Classification

```javascript
// src/components/CreateTicket.jsx
import React, { useState } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const CreateTicket = () => {
  const [formData, setFormData] = useState({
    title: '',
    description: ''
  });
  const [aiSuggestion, setAiSuggestion] = useState(null);

  const API_URL = process.env.REACT_APP_API_URL;
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;

  const classifyWithAI = async () => {
    try {
      const response = await axios.post(
        `${ML_API_URL}/api/ml/classify-ticket`,
        {
          title: formData.title,
          description: formData.description
        }
      );
      
      setAiSuggestion(response.data);
    } catch (error) {
      console.error('AI classification failed:', error);
    }
  };

  const submitTicket = async (e) => {
    e.preventDefault();
    
    try {
      await axios.post(
        `${API_URL}/api/tickets`,
        {
          ...formData,
          category: aiSuggestion?.category || 'general',
          priority: aiSuggestion?.priority || 'medium'
        },
        { headers: getAuthHeader() }
      );
      
      alert('Ticket created successfully!');
      setFormData({ title: '', description: '' });
      setAiSuggestion(null);
    } catch (error) {
      console.error('Failed to create ticket:', error);
    }
  };

  return (
    <div className="create-ticket">
      <form onSubmit={submitTicket}>
        <input
          type="text"
          placeholder="Ticket Title"
          value={formData.title}
          onChange={(e) => setFormData({...formData, title: e.target.value})}
        />
        
        <textarea
          placeholder="Description"
          value={formData.description}
          onChange={(e) => setFormData({...formData, description: e.target.value})}
        />
        
        <button type="button" onClick={classifyWithAI}>
          Get AI Classification
        </button>
        
        {aiSuggestion && (
          <div className="ai-suggestion">
            <p>Category: {aiSuggestion.category}</p>
            <p>Priority: {aiSuggestion.priority}</p>
            <p>Confidence: {(aiSuggestion.confidence * 100).toFixed(1)}%</p>
          </div>
        )}
        
        <button type="submit">Create Ticket</button>
      </form>
    </div>
  );
};

export default CreateTicket;
```

## Backend Implementation Patterns

### Express Route with JWT Authentication

```javascript
// backend/middleware/auth.js
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

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.getTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ 
      assignedTo: req.user.id 
    }).populate('assignedTo', 'name email');
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};
```

### MongoDB Models

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
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
  timeSpent: {
    type: Number,
    default: 0 // minutes
  },
  dueDate: Date
}, {
  timestamps: true
});

module.exports = mongoose.model('Task', taskSchema);
```

## ML Service Implementation

### FastAPI ML Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Dict
import joblib
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, ensemble

app = FastAPI()

# Load or initialize models
try:
    ticket_classifier = joblib.load('./models/ticket_classifier.pkl')
except:
    ticket_classifier = None

risk_detector = ensemble.AdaptiveRandomForestClassifier()
anomaly_detector = anomaly.HalfSpaceTrees()

class TicketInput(BaseModel):
    title: str
    description: str

class RiskInput(BaseModel):
    userId: str
    failedLogins: int
    unusualActivity: bool
    accessPatterns: List[str]

class BurnoutInput(BaseModel):
    userId: str
    tasksCompleted: int
    hoursWorked: float
    overtimeHours: float
    missedDeadlines: int

@app.post("/api/ml/classify-ticket")
async def classify_ticket(data: TicketInput):
    try:
        # Simple rule-based classification (replace with trained model)
        text = f"{data.title} {data.description}".lower()
        
        category = "general"
        if any(word in text for word in ["bug", "error", "broken", "not working"]):
            category = "technical"
        elif any(word in text for word in ["password", "access", "login", "permission"]):
            category = "security"
        elif any(word in text for word in ["feature", "add", "new", "improve"]):
            category = "feature_request"
        
        priority = "medium"
        if any(word in text for word in ["urgent", "critical", "asap", "immediately"]):
            priority = "high"
        elif any(word in text for word in ["minor", "eventually", "low priority"]):
            priority = "low"
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-risk")
async def detect_risk(data: RiskInput):
    try:
        # Calculate risk score
        risk_score = 0.0
        factors = []
        
        if data.failedLogins > 3:
            risk_score += 0.3
            factors.append(f"High failed login attempts: {data.failedLogins}")
        
        if data.unusualActivity:
            risk_score += 0.25
            factors.append("Unusual activity detected")
        
        unusual_patterns = ["night", "weekend"]
        if any(pattern in data.accessPatterns for pattern in unusual_patterns):
            risk_score += 0.2
            factors.append("Unusual access time patterns")
        
        risk_level = "low"
        if risk_score > 0.7:
            risk_level = "high"
        elif risk_score > 0.4:
            risk_level = "medium"
        
        return {
            "riskScore": min(risk_score, 1.0),
            "riskLevel": risk_level,
            "factors": factors
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-burnout")
async def detect_burnout(data: BurnoutInput):
    try:
        # Burnout scoring algorithm
        burnout_score = 0.0
        
        # Hours worked factor
        if data.hoursWorked > 50:
            burnout_score += 0.3
        
        # Overtime factor
        if data.overtimeHours > 10:
            burnout_score += 0.25
        
        # Missed deadlines factor
        if data.missedDeadlines > 2:
            burnout_score += 0.2
        
        # Task completion rate
        if data.tasksCompleted > 40:
            burnout_score += 0.15
        
        recommendation = "monitor"
        if burnout_score > 0.7:
            recommendation = "urgent_intervention"
        elif burnout_score > 0.5:
            recommendation = "reduce_workload"
        
        return {
            "burnoutScore": min(burnout_score, 1.0),
            "recommendation": recommendation,
            "suggestedActions": [
                "Schedule time off",
                "Redistribute tasks",
                "Limit overtime hours"
            ] if burnout_score > 0.5 else []
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-delay")
async def predict_delay(data: dict):
    try:
        # Simple delay prediction
        tasks_remaining = data.get("tasksRemaining", 0)
        avg_time = data.get("averageCompletionTime", 0)
        team_size = data.get("teamSize", 1)
        complexity = data.get("complexityScore", 5)
        
        estimated_days = (tasks_remaining * avg_time) / team_size
        complexity_factor = complexity / 10
        
        delay_probability = min(estimated_days * complexity_factor / 20, 1.0)
        estimated_delay = int(estimated_days * complexity_factor) if delay_probability > 0.5 else 0
        
        return {
            "delayProbability": delay_probability,
            "estimatedDelay": estimated_delay,
            "confidence": 0.75
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Configuration

### Backend Environment Variables

```env
# Server
PORT=5000
NODE_ENV=production

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
# Or MongoDB Atlas: mongodb+srv://user:pass@cluster.mongodb.net/dbname

# Authentication
JWT_SECRET=your_secure_jwt_secret_here
JWT_EXPIRY=24h

# ML Service
ML_SERVICE_URL=http://localhost:8000

# CORS
ALLOWED_ORIGINS=http://localhost:3000,https://yourdomain.com
```

### Frontend Environment Variables

```env
# API URLs
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000

# Production
# REACT_APP_API_URL=https://api.yourdomain.com
# REACT_APP_ML_API_URL=https://ml.yourdomain.com
```

### ML Service Configuration

```python
# ml-service/config.py
import os
from pathlib import Path

class Config:
    MODEL_PATH = Path(os.getenv("MODEL_PATH", "./models"))
    LOG_LEVEL = os.getenv("LOG_LEVEL", "INFO")
    BACKEND_URL = os.getenv("BACKEND_URL", "http://localhost:5000")
    
    # Model parameters
    TICKET_CONFIDENCE_THRESHOLD = 0.7
    RISK_HIGH_THRESHOLD = 0.7
    RISK_MEDIUM_THRESHOLD = 0.4
    BURNOUT_THRESHOLD = 0.6
```

## Common Patterns

### Protected Route Component

```javascript
// src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }
  
  return children;
};

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';

<Routes>
  <Route path="/login" element={<Login />} />
  <Route path="/dashboard" element={
    <ProtectedRoute>
      <Dashboard />
    </ProtectedRoute>
  } />
  <Route path="/admin" element={
    <ProtectedRoute requiredRole="admin">
      <AdminPanel />
    </ProtectedRoute>
  } />
</Routes>
```

### Time Tracking Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId, onSave }) => {
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

  const formatTime = (sec) => {
    const hrs = Math.floor(sec / 3600);
    const mins = Math.floor((sec % 3600) / 60);
    const secs = sec % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const handleSave = () => {
    onSave(Math.floor(seconds / 60)); // Save as minutes
    setSeconds(0);
    setIsRunning(false);
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={handleSave} disabled={seconds === 0}>
        Save Time
      </button>
    </div>
  );
};

export default TimeTracker;
```

### Admin Analytics Dashboard

```javascript
// src/components/AdminAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeTasks: 0,
    openTickets: 0,
    highRiskUsers: []
  });

  const API_URL = process.env.REACT_APP_API_URL;
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const [users, tasks, tickets] = await Promise.all([
        axios.get(`${API_URL}/api/users`, { headers: getAuthHeader() }),
        axios.get(`${API_URL}/api/tasks`, { headers: getAuthHeader() }),
        axios.get(`${API_URL}/api/tickets`, { headers: getAuthHeader() })
      ]);

      // Check for high-risk users
      const riskChecks = await Promise.all(
        users.data.map(user => 
          axios.post(`${ML_API_URL}/api/ml/detect-risk`, {
            userId: user._id,
            failedLogins: user.failedLogins || 0,
            unusualActivity: user.unusualActivity || false,
            accessPatterns: user.accessPatterns || []
          }).catch(() => ({ data: { riskLevel: 'low' } }))
        )
      );

      const highRiskUsers = users.data.filter((user, i) => 
        riskChecks[i].data.riskLevel === 'high'
      );

      setAnalytics({
        totalUsers: users.data.length,
        activeTasks: tasks.data.filter(t => t.status !== 'done').length,
        openTickets: tickets.data.filter(t => t.status !== 'closed').length,
        highRiskUsers
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  return (
    <div className="admin-analytics">
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
      <div className="stat-card alert">
        <h3>High Risk Users</h3>
        <p>{analytics.highRiskUsers.length}</p>
        {analytics.highRiskUsers.length > 0 && (
          <ul>
            {analytics.highRiskUsers.map(user => (
              <li key={user._id}>{user.name} ({user.email})</li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

## Troubleshooting

### JWT Token Expiration

```javascript
// Add axios interceptor to handle token refresh
import axios from 'axios';

axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
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
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Configuration

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');

const app = express();

const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') || 
  ['http://localhost:3000'];

app.use(cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true
}));
```

### ML Service Not Responding

```bash
# Check if ML service is running
curl http://localhost:8000/health

# Check logs
cd ml-service
tail -f logs/app.log

# Restart with verbose logging
uvicorn main:app --reload --log-level debug
```

### Frontend Build Issues

```bash
# Clear cache and rebuild
cd frontend
rm -rf node_modules package-lock.json
npm install
npm start

# Production build
npm run build
```

## Deployment

### Production Build

```bash
# Frontend
cd frontend
npm run build
# Serve build folder with nginx or serve

# Backend
cd backend
npm install --production
NODE_ENV=production node server.js

# ML Service
cd ml-service
pip install -r requirements
