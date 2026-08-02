---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task automation built with React, Node.js, and FastAPI
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement user task tracking with burnout detection"
  - "create admin dashboard with AI insights"
  - "build ticket management system with AI classification"
  - "add anomaly detection to user management"
  - "integrate AI analytics into task management"
  - "configure JWT authentication for enterprise system"
  - "implement Kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that manages users, tasks, and support tickets with integrated AI capabilities. The system provides risk detection, anomaly detection, burnout analysis, and predictive insights to improve organizational productivity and decision-making.

**Architecture:**
- Frontend: React.js with Kanban boards and real-time dashboards
- Backend: Node.js REST APIs with JWT authentication
- ML Service: FastAPI with scikit-learn and River for online learning
- Database: MongoDB for persistent storage

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

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
```bash
PORT=5000
MONGODB_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
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
```bash
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
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
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Core Features

### 1. User Authentication (JWT)

**Backend - User Registration:**
```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    // Create user
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate token
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.status(201).json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

**Authentication Middleware:**
```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ message: 'No token, authorization denied' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Token is not valid' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied: Admin only' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### 2. Task Management with Kanban

**Backend - Task Model:**
```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  status: { 
    type: String, 
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  timeTracked: { type: Number, default: 0 }, // in seconds
  dueDate: { type: Date },
  tags: [String]
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

**Backend - Task Routes:**
```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

// Get all tasks for user
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({
      $or: [
        { assignedTo: req.user.id },
        { createdBy: req.user.id }
      ]
    })
    .populate('assignedTo', 'username email')
    .populate('createdBy', 'username email')
    .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    
    await task.save();
    await task.populate('assignedTo createdBy', 'username email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status },
      { new: true }
    ).populate('assignedTo createdBy', 'username email');
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update time tracked
router.patch('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { timeTracked } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeTracked } },
      { new: true }
    );
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

**Frontend - Kanban Board Component:**
```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], 'in-progress': [], done: [] });
      
      setTasks(grouped);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const handleDrop = (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');
    
    if (currentStatus !== newStatus) {
      updateTaskStatus(taskId, newStatus);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{status.replace('-', ' ').toUpperCase()}</h3>
          <div className="task-list">
            {tasks[status].map(task => (
              <div
                key={task._id}
                className="task-card"
                draggable
                onDragStart={(e) => handleDragStart(e, task._id, status)}
              >
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <div className="task-meta">
                  <span className={`priority ${task.priority}`}>
                    {task.priority}
                  </span>
                  <span className="time">
                    {Math.floor(task.timeTracked / 3600)}h {Math.floor((task.timeTracked % 3600) / 60)}m
                  </span>
                </div>
                {task.assignedTo && (
                  <span className="assigned">@{task.assignedTo.username}</span>
                )}
              </div>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### 3. AI-Powered Ticket Classification

**ML Service - Ticket Classifier:**
```python
# ml-service/models/ticket_classifier.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib
import os

router = APIRouter()

class TicketRequest(BaseModel):
    title: str
    description: str

class TicketResponse(BaseModel):
    category: str
    priority: str
    confidence: float
    suggested_department: str

# Load or initialize model
model_path = os.getenv('MODEL_PATH', './models')
try:
    vectorizer = joblib.load(f'{model_path}/ticket_vectorizer.pkl')
    classifier = joblib.load(f'{model_path}/ticket_classifier.pkl')
except:
    # Initialize with default model
    vectorizer = TfidfVectorizer(max_features=1000)
    classifier = MultinomialNB()

@router.post("/classify", response_model=TicketResponse)
async def classify_ticket(ticket: TicketRequest):
    try:
        # Combine title and description
        text = f"{ticket.title} {ticket.description}"
        
        # Transform text
        features = vectorizer.transform([text])
        
        # Predict category
        category = classifier.predict(features)[0]
        confidence = max(classifier.predict_proba(features)[0])
        
        # Determine priority based on keywords
        priority = determine_priority(text)
        
        # Route to department
        department = route_to_department(category)
        
        return TicketResponse(
            category=category,
            priority=priority,
            confidence=float(confidence),
            suggested_department=department
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def determine_priority(text: str) -> str:
    urgent_keywords = ['urgent', 'critical', 'emergency', 'asap', 'immediately']
    high_keywords = ['important', 'high priority', 'soon', 'blocker']
    
    text_lower = text.lower()
    if any(keyword in text_lower for keyword in urgent_keywords):
        return 'urgent'
    elif any(keyword in text_lower for keyword in high_keywords):
        return 'high'
    else:
        return 'medium'

def route_to_department(category: str) -> str:
    routing_map = {
        'technical': 'IT Support',
        'account': 'Customer Success',
        'billing': 'Finance',
        'feature_request': 'Product Team',
        'bug': 'Engineering',
        'security': 'Security Team'
    }
    return routing_map.get(category, 'General Support')
```

### 4. Burnout Detection

**ML Service - Burnout Analysis:**
```python
# ml-service/models/burnout_detector.py
from fastapi import APIRouter
from pydantic import BaseModel
from typing import List
from datetime import datetime, timedelta
import numpy as np

router = APIRouter()

class UserActivity(BaseModel):
    user_id: str
    tasks_completed: int
    hours_worked: float
    overtime_hours: float
    days_without_break: int
    task_completion_rate: float

class BurnoutResult(BaseModel):
    user_id: str
    burnout_risk: str  # low, medium, high
    risk_score: float
    factors: List[str]
    recommendations: List[str]

@router.post("/analyze", response_model=BurnoutResult)
async def analyze_burnout(activity: UserActivity):
    risk_score = 0.0
    factors = []
    recommendations = []
    
    # Check overtime hours
    if activity.overtime_hours > 10:
        risk_score += 0.3
        factors.append("Excessive overtime hours")
        recommendations.append("Reduce overtime and delegate tasks")
    
    # Check days without break
    if activity.days_without_break > 5:
        risk_score += 0.25
        factors.append("No breaks in extended period")
        recommendations.append("Schedule mandatory rest days")
    
    # Check task completion rate
    if activity.task_completion_rate < 0.6:
        risk_score += 0.2
        factors.append("Low task completion rate")
        recommendations.append("Review workload and prioritize tasks")
    
    # Check hours worked per week
    if activity.hours_worked > 50:
        risk_score += 0.25
        factors.append("Working excessive hours")
        recommendations.append("Set working hour limits")
    
    # Determine risk level
    if risk_score >= 0.7:
        risk_level = "high"
        recommendations.append("Immediate intervention recommended")
    elif risk_score >= 0.4:
        risk_level = "medium"
        recommendations.append("Monitor closely and adjust workload")
    else:
        risk_level = "low"
    
    return BurnoutResult(
        user_id=activity.user_id,
        burnout_risk=risk_level,
        risk_score=min(risk_score, 1.0),
        factors=factors if factors else ["No significant risk factors detected"],
        recommendations=recommendations if recommendations else ["Maintain current work-life balance"]
    )
```

### 5. Anomaly Detection

**ML Service - Anomaly Detection:**
```python
# ml-service/models/anomaly_detector.py
from fastapi import APIRouter
from pydantic import BaseModel
from typing import List, Dict
from river import anomaly, preprocessing
import json

router = APIRouter()

class UserBehavior(BaseModel):
    user_id: str
    login_time: str
    login_location: str
    actions_per_minute: float
    failed_login_attempts: int
    unusual_access_patterns: int

class AnomalyResult(BaseModel):
    is_anomaly: bool
    anomaly_score: float
    risk_level: str
    detected_anomalies: List[str]
    action_required: str

# Initialize online learning anomaly detector
scaler = preprocessing.StandardScaler()
detector = anomaly.HalfSpaceTrees(seed=42)

@router.post("/detect", response_model=AnomalyResult)
async def detect_anomaly(behavior: UserBehavior):
    detected_anomalies = []
    anomaly_score = 0.0
    
    # Feature engineering
    features = {
        'actions_per_minute': behavior.actions_per_minute,
        'failed_login_attempts': behavior.failed_login_attempts,
        'unusual_access_patterns': behavior.unusual_access_patterns,
        'hour_of_day': int(behavior.login_time.split(':')[0])
    }
    
    # Check for immediate red flags
    if behavior.failed_login_attempts > 3:
        detected_anomalies.append("Multiple failed login attempts")
        anomaly_score += 0.4
    
    if behavior.actions_per_minute > 10:
        detected_anomalies.append("Unusually high activity rate (possible bot)")
        anomaly_score += 0.3
    
    if behavior.unusual_access_patterns > 0:
        detected_anomalies.append("Accessing restricted resources")
        anomaly_score += 0.3
    
    # Use ML model for detection
    scaled_features = scaler.learn_one(features).transform_one(features)
    ml_score = detector.score_one(scaled_features)
    detector.learn_one(scaled_features)
    
    # Normalize ML score to 0-1 range
    normalized_ml_score = min(ml_score / 2.0, 1.0)
    anomaly_score = max(anomaly_score, normalized_ml_score)
    
    # Determine risk level and action
    if anomaly_score > 0.7:
        risk_level = "critical"
        action = "Lock account immediately and notify security team"
        is_anomaly = True
    elif anomaly_score > 0.5:
        risk_level = "high"
        action = "Flag for review and monitor closely"
        is_anomaly = True
    elif anomaly_score > 0.3:
        risk_level = "medium"
        action = "Add to watch list"
        is_anomaly = True
    else:
        risk_level = "low"
        action = "No action required"
        is_anomaly = False
    
    return AnomalyResult(
        is_anomaly=is_anomaly,
        anomaly_score=round(anomaly_score, 3),
        risk_level=risk_level,
        detected_anomalies=detected_anomalies if detected_anomalies else ["No anomalies detected"],
        action_required=action
    )
```

### 6. Admin Dashboard with Analytics

**Backend - Analytics Endpoint:**
```javascript
// backend/routes/analytics.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const User = require('../models/User');
const Ticket = require('../models/Ticket');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

router.get('/dashboard', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    // User statistics
    const totalUsers = await User.countDocuments();
    const activeUsers = await User.countDocuments({ 
      lastActive: { $gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) }
    });
    
    // Task statistics
    const taskStats = await Task.aggregate([
      {
        $group: {
          _id: '$status',
          count: { $sum: 1 },
          avgTimeTracked: { $avg: '$timeTracked' }
        }
      }
    ]);
    
    // Ticket statistics
    const ticketStats = await Ticket.aggregate([
      {
        $group: {
          _id: '$status',
          count: { $sum: 1 }
        }
      }
    ]);
    
    // Workload distribution
    const workloadDistribution = await Task.aggregate([
      {
        $match: { status: { $in: ['todo', 'in-progress'] } }
      },
      {
        $group: {
          _id: '$assignedTo',
          taskCount: { $sum: 1 },
          totalTimeTracked: { $sum: '$timeTracked' }
        }
      },
      {
        $lookup: {
          from: 'users',
          localField: '_id',
          foreignField: '_id',
          as: 'user'
        }
      },
      {
        $unwind: '$user'
      },
      {
        $project: {
          username: '$user.username',
          taskCount: 1,
          totalTimeTracked: 1,
          avgTimePerTask: { $divide: ['$totalTimeTracked', '$taskCount'] }
        }
      }
    ]);
    
    res.json({
      users: { total: totalUsers, active: activeUsers },
      tasks: taskStats,
      tickets: ticketStats,
      workload: workloadDistribution
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## Configuration

### Backend Configuration (`backend/.env`)
```bash
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise-ums

# Authentication
JWT_SECRET=your_secure_random_string_here
JWT_EXPIRATION=24h

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (for notifications)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
```

### ML Service Configuration (`ml-service/.env`)
```bash
# Backend API
BACKEND_URL=http://localhost:5000

# Model storage
MODEL_PATH=./models

# ML Settings
CONFIDENCE_THRESHOLD=0.75
ANOMALY_THRESHOLD=0.5
```

### Frontend Configuration (`frontend/.env`)
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_ENABLE_AI_FEATURES=true
```

## Common Patterns

### API Authentication Pattern
```javascript
// frontend/src/utils/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

// Add token to all requests
api.interceptors.request.use(
  config => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  error => Promise.reject(error)
);

// Handle token expiration
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

### Time Tracking Pattern
```javascript
// frontend/src/hooks/useTimeTracker.js
import { useState, useEffect, useRef } from 'react';
import api from '../utils/api';

const useTimeTracker = (taskId) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else {
      if (intervalRef.current) clearInterval(intervalRef.current);
    }
    
    return () => {
      if (intervalRef.current) clearInterval(intervalRef.current);
    };
  }, [isRunning]);

  const start = () => setIsRunning(true);
  
  const pause = () => setIsRunning(false);
  
  const stop = async () => {
    setIsRunning(false);
    if (seconds > 0) {
      await api.patch(`/api/tasks/${taskId}/time`, { timeTracked: seconds });
      setSeconds(0);
    }
  };

  return { seconds, isRunning, start, pause, stop };
};

export default useTimeTracker;
```

### AI Integration Pattern
```javascript
// backend/services/aiService.js
const axios = require('axios');

class AIService {
  constructor() {
    this.baseURL = process.env.ML_SERVICE_URL;
  }

  async classifyTicket(title, description) {
    try {
      const response = await axios.post(`${this.baseURL}/tickets/classify`, {
        title,
        description
      });
      return response.data;
    } catch (error) {
      console.error('AI classification error:', error);
      return { category: 'general', priority: 'medium', confidence: 0 };
    }
  }

  async detectBurnout(userId, activityData) {
    try {
      const response = await axios.post(`${this.baseURL}/burnout/analyze`, {
        user_id: userId,
        ...activityData
      });
      return response.data;
    } catch (error) {
      console.error('Burnout detection error:', error);
      return { burnout_risk: 'unknown', risk_score: 0 };
    }
  }

  async detectAnomaly(userBehavior) {
    try {
      const response = await axios.post(`${this.baseURL}/anomaly/detect`, userBehavior);
      return response.data;
    } catch (error) {
      console.error('Anomaly detection error:', error);
      return { is_anomaly: false, risk_level: 'unknown' };
    }
  }
}

module.exports = new AIService();
```

## Troubleshooting

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
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    // Retry connection after 5 seconds
    setTimeout(connectDB, 5000);
  }
};

module.exports = connectDB;
```

### CORS Issues
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Model Not Loading
```python
# ml-service/utils/model_loader.py
import os
import joblib
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB

def load_or_create_model(model_path):
    vectorizer_path = f"{model_path}/ticket_vectorizer.pkl"
    classifier_path = f"{model_path}/ticket_classifier.pkl"
    
    if os.path.exists(vectorizer_path) and os.path.exists(classifier_path):
        print("Loading existing models...")
        vectorizer = joblib.load(vectorizer_path)
        classifier = joblib.load(classifier_path)
    else:
        print("Creating new models...")
        os.makedirs(model_path, exist_ok=True)
        vectorizer = TfidfVectorizer(max_features=1000)
        classifier = MultinomialNB()
        
        # Train with minimal data
        sample_texts = ["technical issue",
