---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, risk detection, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement task management with burnout detection"
  - "create admin dashboard with AI insights"
  - "build user management system with ML features"
  - "configure AI-powered ticket classification"
  - "develop Kanban board with time tracking"
  - "implement anomaly detection for user behavior"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System, a full-stack application that combines user/task management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Role-based access control, authentication with JWT, user CRUD operations
- **Task Management**: Kanban board (To Do → In Progress → Done), time tracking, task assignment
- **Support Tickets**: Ticket creation, tracking, and AI-based classification/routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout detection, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, user monitoring, alerts

**Architecture**: React.js frontend + Node.js/Express backend + FastAPI ML service + MongoDB database

## Installation & Setup

### Prerequisites

```bash
# Ensure you have installed:
# - Node.js (v14+)
# - Python (3.8+)
# - MongoDB (running instance)
```

### Clone and Install

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt

# Install frontend dependencies
cd ../frontend
npm install
```

### Environment Configuration

**Backend** (`backend/.env`):
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend** (`frontend/.env`):
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

**ML Service** (`ml-service/.env`):
```bash
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

### Running the Application

```bash
# Terminal 1: Start MongoDB (if not running as service)
mongod

# Terminal 2: Start Backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 3: Start ML Service
cd ml-service
uvicorn main:app --reload
# Runs at http://localhost:8000

# Terminal 4: Start Frontend
cd frontend
npm start
# Runs at http://localhost:3000
```

## Backend API Reference

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Check if user exists
    let user = await User.findOne({ email });
    if (user) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    // Create user
    user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate JWT
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ token, user: { id: user._id, username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    // Find user
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    // Verify password
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    // Generate JWT
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const auth = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No token, authorization denied' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Token is not valid' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { auth, adminOnly };
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { auth } = require('../middleware/auth');

// Get all tasks for user
router.get('/', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Create new task
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, priority, dueDate, assignedTo } = req.body;
    
    const task = new Task({
      title,
      description,
      priority: priority || 'medium',
      status: 'todo',
      dueDate,
      assignedTo,
      assignedBy: req.user.id,
      timeTracked: 0
    });
    
    await task.save();
    await task.populate('assignedTo assignedBy', 'username email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.status = status;
    if (status === 'done' && !task.completedAt) {
      task.completedAt = new Date();
    }
    
    await task.save();
    await task.populate('assignedTo assignedBy', 'username email');
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Track time on task
router.patch('/:id/time', auth, async (req, res) => {
  try {
    const { seconds } = req.body;
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.timeTracked = (task.timeTracked || 0) + seconds;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Support Tickets

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { auth } = require('../middleware/auth');

// Create support ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    let category = 'general';
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        title,
        description
      });
      category = mlResponse.data.category;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      priority: priority || 'medium',
      category,
      status: 'open',
      createdBy: req.user.id
    });
    
    await ticket.save();
    await ticket.populate('createdBy', 'username email');
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Get user tickets
router.get('/', auth, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .populate('createdBy assignedTo', 'username email')
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### User Management (Admin)

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { auth, adminOnly } = require('../middleware/auth');

// Get all users (admin only)
router.get('/', auth, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update user role (admin only)
router.patch('/:id/role', auth, adminOnly, async (req, res) => {
  try {
    const { role } = req.body;
    
    const user = await User.findById(req.params.id);
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    user.role = role;
    await user.save();
    
    res.json({ message: 'User role updated', user: user.toObject({ versionKey: false, password: 0 }) });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Delete user (admin only)
router.delete('/:id', auth, adminOnly, async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

## ML Service (FastAPI)

### Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from river import anomaly, linear_model
import pickle
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models (load or initialize)
ticket_classifier = None
vectorizer = None
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, seed=42)
burnout_model = linear_model.LogisticRegression()

# Request models
class TicketClassifyRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    task_completion_rate: float
    avg_task_time: float
    overdue_tasks: int
    ticket_count: int

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    login_time: str
    actions_count: int
    failed_logins: int
    ip_address: str

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_working_hours: float
    overtime_hours: float

# Initialize or load models
@app.on_event("startup")
async def load_models():
    global ticket_classifier, vectorizer
    
    model_path = os.getenv("MODEL_PATH", "./models")
    os.makedirs(model_path, exist_ok=True)
    
    try:
        with open(f"{model_path}/ticket_classifier.pkl", "rb") as f:
            ticket_classifier = pickle.load(f)
        with open(f"{model_path}/vectorizer.pkl", "rb") as f:
            vectorizer = pickle.load(f)
    except FileNotFoundError:
        # Train simple model with dummy data
        vectorizer = TfidfVectorizer(max_features=100)
        ticket_classifier = MultinomialNB()
        
        # Dummy training data
        texts = [
            "password reset issue", "cannot access dashboard", "bug in report",
            "new feature request", "performance is slow", "security vulnerability"
        ]
        labels = ["technical", "technical", "bug", "feature", "performance", "security"]
        
        X = vectorizer.fit_transform(texts)
        ticket_classifier.fit(X, labels)
        
        # Save models
        with open(f"{model_path}/ticket_classifier.pkl", "wb") as f:
            pickle.dump(ticket_classifier, f)
        with open(f"{model_path}/vectorizer.pkl", "wb") as f:
            pickle.dump(vectorizer, f)

@app.get("/")
async def root():
    return {"message": "Enterprise User Management ML Service", "status": "running"}

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassifyRequest):
    try:
        text = f"{request.title} {request.description}"
        X = vectorizer.transform([text])
        category = ticket_classifier.predict(X)[0]
        probabilities = ticket_classifier.predict_proba(X)[0]
        
        return {
            "category": category,
            "confidence": float(max(probabilities)),
            "all_probabilities": {
                cls: float(prob) 
                for cls, prob in zip(ticket_classifier.classes_, probabilities)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Risk score calculation based on metrics
        risk_score = 0.0
        
        # Low completion rate increases risk
        if request.task_completion_rate < 0.7:
            risk_score += (0.7 - request.task_completion_rate) * 50
        
        # High average task time increases risk
        if request.avg_task_time > 24:  # hours
            risk_score += (request.avg_task_time - 24) * 2
        
        # Overdue tasks significantly increase risk
        risk_score += request.overdue_tasks * 10
        
        # High ticket count may indicate issues
        if request.ticket_count > 5:
            risk_score += (request.ticket_count - 5) * 5
        
        # Normalize to 0-100
        risk_score = min(100, risk_score)
        
        risk_level = "low"
        if risk_score > 70:
            risk_level = "high"
        elif risk_score > 40:
            risk_level = "medium"
        
        return {
            "user_id": request.user_id,
            "risk_score": round(risk_score, 2),
            "risk_level": risk_level,
            "factors": {
                "completion_rate": request.task_completion_rate,
                "avg_task_time": request.avg_task_time,
                "overdue_tasks": request.overdue_tasks
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        # Create feature vector
        features = {
            "actions_count": request.actions_count,
            "failed_logins": request.failed_logins,
            "hour": int(request.login_time.split(":")[0]) if ":" in request.login_time else 12
        }
        
        # Get anomaly score
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.5
        
        return {
            "user_id": request.user_id,
            "is_anomaly": is_anomaly,
            "anomaly_score": round(score, 4),
            "details": {
                "actions_count": request.actions_count,
                "failed_logins": request.failed_logins,
                "login_time": request.login_time
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        burnout_score = 0.0
        
        # High workload
        if request.tasks_assigned > 20:
            burnout_score += (request.tasks_assigned - 20) * 2
        
        # Low completion rate
        completion_rate = request.tasks_completed / max(request.tasks_assigned, 1)
        if completion_rate < 0.7:
            burnout_score += (0.7 - completion_rate) * 50
        
        # Long working hours
        if request.avg_working_hours > 8:
            burnout_score += (request.avg_working_hours - 8) * 5
        
        # Overtime
        burnout_score += request.overtime_hours * 3
        
        # Normalize
        burnout_score = min(100, burnout_score)
        
        burnout_level = "low"
        if burnout_score > 70:
            burnout_level = "high"
        elif burnout_score > 40:
            burnout_level = "medium"
        
        return {
            "user_id": request.user_id,
            "burnout_score": round(burnout_score, 2),
            "burnout_level": burnout_level,
            "recommendations": get_burnout_recommendations(burnout_level),
            "metrics": {
                "tasks_assigned": request.tasks_assigned,
                "completion_rate": round(completion_rate, 2),
                "avg_working_hours": request.avg_working_hours,
                "overtime_hours": request.overtime_hours
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_burnout_recommendations(level: str) -> List[str]:
    recommendations = {
        "low": ["Maintain current workload", "Continue regular breaks"],
        "medium": [
            "Consider redistributing some tasks",
            "Schedule regular breaks",
            "Monitor workload closely"
        ],
        "high": [
            "Urgent: Reduce task assignment immediately",
            "Mandatory break period recommended",
            "Consider temporary workload reassignment",
            "Schedule one-on-one discussion"
        ]
    }
    return recommendations.get(level, [])
```

## Frontend Integration

### API Service Setup

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_URL = process.env.REACT_APP_ML_SERVICE_URL || 'http://localhost:8000';

// Create axios instances
const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

const mlApi = axios.create({
  baseURL: ML_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add auth token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auth services
export const authService = {
  login: (email, password) => api.post('/api/auth/login', { email, password }),
  register: (userData) => api.post('/api/auth/register', userData),
  logout: () => localStorage.removeItem('token')
};

// Task services
export const taskService = {
  getTasks: () => api.get('/api/tasks'),
  createTask: (taskData) => api.post('/api/tasks', taskData),
  updateTaskStatus: (id, status) => api.patch(`/api/tasks/${id}/status`, { status }),
  trackTime: (id, seconds) => api.patch(`/api/tasks/${id}/time`, { seconds }),
  deleteTask: (id) => api.delete(`/api/tasks/${id}`)
};

// Ticket services
export const ticketService = {
  getTickets: () => api.get('/api/tickets'),
  createTicket: (ticketData) => api.post('/api/tickets', ticketData),
  updateTicket: (id, updates) => api.patch(`/api/tickets/${id}`, updates)
};

// User services (admin)
export const userService = {
  getUsers: () => api.get('/api/users'),
  updateUserRole: (id, role) => api.patch(`/api/users/${id}/role`, { role }),
  deleteUser: (id) => api.delete(`/api/users/${id}`)
};

// ML services
export const mlService = {
  classifyTicket: (title, description) => 
    mlApi.post('/classify-ticket', { title, description }),
  predictRisk: (userData) => 
    mlApi.post('/predict-risk', userData),
  detectAnomaly: (activityData) => 
    mlApi.post('/detect-anomaly', activityData),
  detectBurnout: (workloadData) => 
    mlApi.post('/detect-burnout', workloadData)
};

export default api;
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskService.getTasks();
      const tasksByStatus = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(tasksByStatus);
      setLoading(false);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
      setLoading(false);
    }
  };

  const handleDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');

    if (currentStatus === newStatus) return;

    try {
      await taskService.updateTaskStatus(taskId, newStatus);
      await fetchTasks();
    } catch (error) {
      console.error('Failed to update task status:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const renderColumn = (status, title, taskList) => (
    <div 
      className="kanban-column"
      onDrop={(e) => handleDrop(e, status)}
      onDragOver={handleDragOver}
    >
      <h3>{title} ({taskList.length})</h3>
      <div className="task-list">
        {taskList.map(task => (
          <div
            key={task._id}
            className={`task-card priority-${task.priority}`}
            draggable
            onDragStart={(e) => handleDragStart(e, task._id, status)}
          >
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <div className="task-meta">
              <span className="priority">{task.priority}</span>
              {task.dueDate && (
                <span className="due-date">
                  Due: {new Date(task.dueDate).toLocaleDateString()}
                </span>
              )}
            </div>
            {task.timeTracked > 0 && (
              <div className="time-tracked">
                ⏱ {Math.floor(task.timeTracked / 3600)}h {Math.floor((task.timeTracked % 3600) / 60)}m
              </div>
            )}
          </div>
        ))}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do', tasks.todo)}
      {renderColumn('in-progress', 'In Progress', tasks.inProgress)}
      {renderColumn('done', 'Done', tasks.done)}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { mlService, userService } from '../services/api';
import './AIAnalytics.css';

const AIAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    burnoutUsers: [],
    anomalies: []
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // This would typically fetch user metrics from backend
      // and call ML service for predictions
      const usersResponse = await userService.getUsers();
      const users = usersResponse.data;

      // Example: Analyze each user
      const riskPredictions = await Promise.all(
        users.slice(0, 5).map(user => 
          mlService.predictRisk({
            user_id: user._id,
            task_completion_rate: Math.random(),
            avg_task_time: Math.random() * 48,
            overdue_tasks: Math.floor(Math.random() * 10),
            ticket_count: Math.floor(Math.random() * 8)
          }).catch(e => null)
        )
      );

      const burnoutPredictions = await Promise.all(
        users.slice(0, 5).map(user =>
          mlService.detectBurnout({
            user_id: user._id,
            tasks_assigned: Math.floor(Math.random() * 30),
            tasks_completed: Math.floor(Math.random() * 25),
            avg_working_hours: 6 + Math.random() * 6,
            overtime_hours: Math.random() * 20
          }).catch(e => null)
        )
      );

      setAnalytics({
        riskUsers: riskPredictions.filter(r => r && r.data.risk_level !== 'low'),
        burnoutUsers: burnoutPredictions.filter(b => b && b.data.burnout_level !== 'low'),
        anomalies: []
      });
      setLoading(false);
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Loading AI analytics...</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Analytics</h2>

      <div className
