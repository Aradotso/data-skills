---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user task tracking with burnout detection"
  - "build admin dashboard with role-based access"
  - "integrate ML service for ticket classification"
  - "configure user management with AI insights"
  - "add anomaly detection to user system"
  - "setup kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack JavaScript application that combines user/task management with AI-powered analytics. It provides role-based access control, Kanban-style task tracking, support ticket management, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and project delay forecasting.

**Key Components:**
- **Frontend**: React.js dashboard for users and admins
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based AI analytics using scikit-learn and River
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

```bash
# Required tools
node >= 14.x
npm >= 6.x
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

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
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

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
# Frontend runs at http://localhost:3000
```

## Backend API Usage

### Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'Authentication required' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

module.exports = authMiddleware;
```

### User Management Routes

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const authMiddleware = require('../middleware/auth');
const adminMiddleware = require('../middleware/admin');

// Register user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }
    
    const user = new User({ username, email, password, role: role || 'user' });
    await user.save();
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ user, token });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    
    if (!user || !(await user.comparePassword(password))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ user, token });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get all users (Admin only)
router.get('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update user
router.put('/:id', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const { username, email, role, status } = req.body;
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { username, email, role, status },
      { new: true }
    ).select('-password');
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Delete user
router.delete('/:id', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    await User.findByIdAndDelete(req.params.id);
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const authMiddleware = require('../middleware/auth');

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      createdBy: req.user.userId,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tasks
router.get('/my-tasks', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status (for Kanban)
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body; // 'todo', 'inprogress', 'done'
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time on task
router.post('/:id/time-log', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body; // duration in seconds
    
    const task = await Task.findById(req.params.id);
    task.timeSpent = (task.timeSpent || 0) + duration;
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
const authMiddleware = require('../middleware/auth');
const axios = require('axios');

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Get AI classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      { title, description }
    );
    
    const ticket = new Ticket({
      title,
      description,
      createdBy: req.user.userId,
      priority: priority || mlResponse.data.priority,
      category: mlResponse.data.category,
      status: 'open',
      assignedTo: mlResponse.data.suggestedAssignee
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get all tickets
router.get('/', authMiddleware, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.userId };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update ticket
router.put('/:id', authMiddleware, async (req, res) => {
  try {
    const { status, assignedTo, priority, resolution } = req.body;
    
    const ticket = await Ticket.findByIdAndUpdate(
      req.params.id,
      { status, assignedTo, priority, resolution, updatedAt: Date.now() },
      { new: true }
    );
    
    res.json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service API

### Ticket Classification

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
    title: str
    description: str

class TicketClassification(BaseModel):
    category: str
    priority: str
    suggestedAssignee: str
    confidence: float

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

try:
    vectorizer = joblib.load(f'{MODEL_PATH}/vectorizer.pkl')
    category_model = joblib.load(f'{MODEL_PATH}/category_model.pkl')
    priority_model = joblib.load(f'{MODEL_PATH}/priority_model.pkl')
except:
    # Initialize new models if not found
    vectorizer = TfidfVectorizer(max_features=1000)
    category_model = MultinomialNB()
    priority_model = MultinomialNB()

@app.post("/classify-ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketRequest):
    try:
        text = f"{ticket.title} {ticket.description}"
        features = vectorizer.transform([text])
        
        category = category_model.predict(features)[0]
        category_proba = category_model.predict_proba(features)[0].max()
        
        priority = priority_model.predict(features)[0]
        
        # Simple assignee logic based on category
        assignee_map = {
            'technical': '507f1f77bcf86cd799439011',
            'billing': '507f1f77bcf86cd799439012',
            'general': '507f1f77bcf86cd799439013'
        }
        
        return TicketClassification(
            category=category,
            priority=priority,
            suggestedAssignee=assignee_map.get(category, assignee_map['general']),
            confidence=float(category_proba)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Anomaly Detection

```python
# ml-service/anomaly_detection.py
from fastapi import APIRouter
from pydantic import BaseModel
from river import anomaly
from river import compose
from river import preprocessing
import datetime

router = APIRouter()

class UserActivity(BaseModel):
    userId: str
    loginTime: str
    location: str
    ipAddress: str
    actionsPerHour: int

class AnomalyResult(BaseModel):
    isAnomaly: bool
    score: float
    reason: str

# Initialize online learning anomaly detector
anomaly_detector = compose.Pipeline(
    preprocessing.StandardScaler(),
    anomaly.HalfSpaceTrees(seed=42)
)

@router.post("/detect-anomaly", response_model=AnomalyResult)
async def detect_anomaly(activity: UserActivity):
    try:
        # Convert activity to features
        hour = datetime.datetime.fromisoformat(activity.loginTime).hour
        features = {
            'hour': hour,
            'actions_per_hour': activity.actionsPerHour,
            'is_weekend': 1 if datetime.datetime.fromisoformat(activity.loginTime).weekday() >= 5 else 0
        }
        
        # Get anomaly score
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.7
        reason = ""
        
        if is_anomaly:
            if activity.actionsPerHour > 100:
                reason = "Unusually high activity rate"
            elif hour < 6 or hour > 22:
                reason = "Login at unusual hours"
            else:
                reason = "Unusual behavior pattern detected"
        
        return AnomalyResult(
            isAnomaly=is_anomaly,
            score=float(score),
            reason=reason
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Burnout Detection

```python
# ml-service/burnout_analysis.py
from fastapi import APIRouter
from pydantic import BaseModel
from typing import List

router = APIRouter()

class TaskLoad(BaseModel):
    userId: str
    tasksInProgress: int
    hoursWorkedThisWeek: float
    tasksOverdue: int
    avgTaskCompletionTime: float

class BurnoutRisk(BaseModel):
    riskLevel: str  # 'low', 'medium', 'high'
    score: float
    recommendations: List[str]

@router.post("/analyze-burnout", response_model=BurnoutRisk)
async def analyze_burnout(load: TaskLoad):
    try:
        score = 0
        recommendations = []
        
        # Calculate burnout score
        if load.hoursWorkedThisWeek > 50:
            score += 30
            recommendations.append("Reduce weekly working hours")
        
        if load.tasksInProgress > 10:
            score += 25
            recommendations.append("Delegate or postpone some tasks")
        
        if load.tasksOverdue > 3:
            score += 25
            recommendations.append("Prioritize overdue tasks")
        
        if load.avgTaskCompletionTime > 8:  # hours
            score += 20
            recommendations.append("Break down complex tasks into smaller chunks")
        
        # Determine risk level
        if score < 30:
            risk_level = 'low'
        elif score < 60:
            risk_level = 'medium'
            recommendations.append("Consider taking a break")
        else:
            risk_level = 'high'
            recommendations.append("Immediate intervention needed - contact manager")
        
        return BurnoutRisk(
            riskLevel=risk_level,
            score=float(score),
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Project Delay Prediction

```python
# ml-service/project_insights.py
from fastapi import APIRouter
from pydantic import BaseModel
from sklearn.ensemble import RandomForestClassifier
import numpy as np

router = APIRouter()

class ProjectData(BaseModel):
    totalTasks: int
    completedTasks: int
    daysRemaining: int
    teamSize: int
    avgTaskComplexity: float  # 1-10 scale

class DelayPrediction(BaseModel):
    willDelay: bool
    probability: float
    estimatedDelay: int  # days
    suggestions: List[str]

# Simple pre-trained model (in production, load from file)
delay_predictor = RandomForestClassifier(n_estimators=100, random_state=42)

@router.post("/predict-delay", response_model=DelayPrediction)
async def predict_delay(project: ProjectData):
    try:
        # Calculate features
        completion_rate = project.completedTasks / project.totalTasks if project.totalTasks > 0 else 0
        tasks_per_day_needed = (project.totalTasks - project.completedTasks) / project.daysRemaining if project.daysRemaining > 0 else 999
        team_capacity = project.teamSize * 3  # Assume 3 tasks per person per day
        
        features = np.array([[
            completion_rate,
            tasks_per_day_needed,
            project.teamSize,
            project.avgTaskComplexity,
            project.daysRemaining
        ]])
        
        # Simple heuristic prediction
        will_delay = tasks_per_day_needed > team_capacity
        probability = min(tasks_per_day_needed / team_capacity, 1.0) if will_delay else 0.2
        
        estimated_delay = 0
        suggestions = []
        
        if will_delay:
            estimated_delay = int((tasks_per_day_needed - team_capacity) * project.daysRemaining / team_capacity)
            suggestions.append(f"Consider adding {int(tasks_per_day_needed / 3 - project.teamSize)} more team members")
            suggestions.append("Reduce scope or extend deadline")
            suggestions.append("Automate repetitive tasks")
        else:
            suggestions.append("Project on track - maintain current pace")
        
        return DelayPrediction(
            willDelay=will_delay,
            probability=float(probability),
            estimatedDelay=estimated_delay,
            suggestions=suggestions
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend Integration

### React API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_URL
});

const mlApi = axios.create({
  baseURL: ML_API_URL
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authService = {
  login: (email, password) => api.post('/users/login', { email, password }),
  register: (userData) => api.post('/users/register', userData),
  getCurrentUser: () => api.get('/users/me')
};

export const userService = {
  getAll: () => api.get('/users'),
  update: (id, data) => api.put(`/users/${id}`, data),
  delete: (id) => api.delete(`/users/${id}`)
};

export const taskService = {
  getMyTasks: () => api.get('/tasks/my-tasks'),
  create: (taskData) => api.post('/tasks', taskData),
  updateStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  logTime: (id, duration) => api.post(`/tasks/${id}/time-log`, { duration })
};

export const ticketService = {
  getAll: () => api.get('/tickets'),
  create: (ticketData) => api.post('/tickets', ticketData),
  update: (id, data) => api.put(`/tickets/${id}`, data)
};

export const mlService = {
  detectAnomaly: (activityData) => mlApi.post('/detect-anomaly', activityData),
  analyzeBurnout: (loadData) => mlApi.post('/analyze-burnout', loadData),
  predictDelay: (projectData) => mlApi.post('/predict-delay', projectData)
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
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskService.getMyTasks();
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inprogress: response.data.filter(t => t.status === 'inprogress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
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

    if (currentStatus !== newStatus) {
      try {
        await taskService.updateStatus(taskId, newStatus);
        fetchTasks();
      } catch (error) {
        console.error('Error updating task:', error);
      }
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="kanban-board">
      <div className="kanban-column"
           onDrop={(e) => handleDrop(e, 'todo')}
           onDragOver={handleDragOver}>
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => (
          <div key={task._id}
               className="task-card"
               draggable
               onDragStart={(e) => handleDragStart(e, task._id, 'todo')}>
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority ${task.priority}`}>{task.priority}</span>
          </div>
        ))}
      </div>

      <div className="kanban-column"
           onDrop={(e) => handleDrop(e, 'inprogress')}
           onDragOver={handleDragOver}>
        <h3>In Progress ({tasks.inprogress.length})</h3>
        {tasks.inprogress.map(task => (
          <div key={task._id}
               className="task-card"
               draggable
               onDragStart={(e) => handleDragStart(e, task._id, 'inprogress')}>
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority ${task.priority}`}>{task.priority}</span>
          </div>
        ))}
      </div>

      <div className="kanban-column"
           onDrop={(e) => handleDrop(e, 'done')}
           onDragOver={handleDragOver}>
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => (
          <div key={task._id}
               className="task-card"
               draggable
               onDragStart={(e) => handleDragStart(e, task._id, 'done')}>
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority ${task.priority}`}>{task.priority}</span>
          </div>
        ))}
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
import { mlService, taskService } from '../services/api';

const AIAnalytics = ({ userId }) => {
  const [burnoutRisk, setBurnoutRisk] = useState(null);
  const [anomalies, setAnomalies] = useState([]);
  const [projectPrediction, setProjectPrediction] = useState(null);

  useEffect(() => {
    analyzeUserBurnout();
    checkAnomalies();
  }, [userId]);

  const analyzeUserBurnout = async () => {
    try {
      const tasks = await taskService.getMyTasks();
      const loadData = {
        userId,
        tasksInProgress: tasks.data.filter(t => t.status === 'inprogress').length,
        hoursWorkedThisWeek: calculateWeeklyHours(tasks.data),
        tasksOverdue: tasks.data.filter(t => new Date(t.dueDate) < new Date()).length,
        avgTaskCompletionTime: calculateAvgTime(tasks.data)
      };

      const response = await mlService.analyzeBurnout(loadData);
      setBurnoutRisk(response.data);
    } catch (error) {
      console.error('Error analyzing burnout:', error);
    }
  };

  const checkAnomalies = async () => {
    try {
      const activityData = {
        userId,
        loginTime: new Date().toISOString(),
        location: 'Office',
        ipAddress: '192.168.1.1',
        actionsPerHour: 45
      };

      const response = await mlService.detectAnomaly(activityData);
      if (response.data.isAnomaly) {
        setAnomalies(prev => [...prev, response.data]);
      }
    } catch (error) {
      console.error('Error checking anomalies:', error);
    }
  };

  const calculateWeeklyHours = (tasks) => {
    const weekAgo = new Date();
    weekAgo.setDate(weekAgo.getDate() - 7);
    
    return tasks
      .filter(t => new Date(t.createdAt) > weekAgo)
      .reduce((sum, t) => sum + (t.timeSpent || 0), 0) / 3600;
  };

  const calculateAvgTime = (tasks) => {
    const completed = tasks.filter(t => t.status === 'done');
    if (completed.length === 0) return 0;
    
    const totalTime = completed.reduce((sum, t) => sum + (t.timeSpent || 0), 0);
    return (totalTime / completed.length) / 3600;
  };

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>

      {burnoutRisk && (
        <div className={`burnout-card ${burnoutRisk.riskLevel}`}>
          <h3>Burnout Risk: {burnoutRisk.riskLevel.toUpperCase()}</h3>
          <p>Score: {burnoutRisk.score.toFixed(1)}/100</p>
          <div className="recommendations">
            <h4>Recommendations:</h4>
            <ul>
              {burnoutRisk.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        </div>
      )}

      {anomalies.length > 0 && (
        <div className="anomaly-alerts">
          <h3>Security Alerts</h3>
          {anomalies.map((anomaly, idx) => (
            <div key={idx} className="alert-card">
              <p><strong>Anomaly Detected:</strong> {anomaly.reason}</p>
              <p>Confidence: {(anomaly.score * 100).toFixed(1)}%</p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user
