---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and automated insights
triggers:
  - "help me build a user management system with AI analytics"
  - "how do I implement AI-powered task management"
  - "set up enterprise user administration with ML insights"
  - "create a support ticket system with AI classification"
  - "build a Kanban board with burnout detection"
  - "integrate AI analytics into user management"
  - "implement JWT authentication with role-based access"
  - "add anomaly detection to user activity tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This is a comprehensive full-stack application that combines user management, task tracking, and support ticketing with AI-powered analytics. The system provides risk detection, anomaly identification, burnout analysis, and predictive project insights using machine learning models.

**Architecture:**
- **Frontend:** React.js for admin and user dashboards
- **Backend:** Node.js with Express/REST APIs
- **ML Service:** FastAPI with scikit-learn and River for online learning
- **Database:** MongoDB for persistent storage
- **Auth:** JWT-based authentication with role-based access control

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
python >= 3.8
mongodb >= 4.x
```

### Setup All Services

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env  # Configure environment variables
npm start  # Runs on http://localhost:5000

# ML Service setup
cd ../ml-service
pip install -r requirements.txt
cp .env.example .env
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup
cd ../frontend
npm install
cp .env.example .env
npm start  # Runs on http://localhost:3000
```

### Environment Configuration

**Backend (.env):**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env):**
```bash
PORT=8000
MODEL_PATH=./models
RETRAIN_INTERVAL=86400
LOG_LEVEL=INFO
```

**Frontend (.env):**
```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

## Key Features & API Endpoints

### Authentication & User Management

#### User Registration & Login

```javascript
// Backend API: POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// Backend API: POST /api/auth/login
const loginUser = async (credentials) => {
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

#### Protected Route Middleware

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

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Task Management with Kanban

#### Creating and Managing Tasks

```javascript
// Backend API: POST /api/tasks
const createTask = async (taskData, token) => {
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
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'in-progress', 'done'
      deadline: taskData.deadline
    })
  });
  return response.json();
};

// Backend API: PUT /api/tasks/:id/status
const updateTaskStatus = async (taskId, newStatus, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};

// Backend API: GET /api/tasks/user/:userId
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

#### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId, token }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  
  useEffect(() => {
    fetchTasks();
  }, [userId]);
  
  const fetchTasks = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    });
  };
  
  const handleDrop = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };
  
  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <div key={task._id} draggable onDragEnd={() => handleDrop(task._id, 'todo')}>
            {task.title}
          </div>
        ))}
      </div>
      <div className="kanban-column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <div key={task._id} draggable onDragEnd={() => handleDrop(task._id, 'in-progress')}>
            {task.title}
          </div>
        ))}
      </div>
      <div className="kanban-column">
        <h3>Done</h3>
        {tasks.done.map(task => (
          <div key={task._id}>{task.title}</div>
        ))}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### Support Ticket System

#### Creating and Managing Tickets

```javascript
// Backend API: POST /api/tickets
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category,
      priority: ticketData.priority,
      createdBy: ticketData.userId
    })
  });
  return response.json();
};

// Backend API: GET /api/tickets
const getAllTickets = async (token, filters = {}) => {
  const queryParams = new URLSearchParams(filters).toString();
  const response = await fetch(`http://localhost:5000/api/tickets?${queryParams}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Backend API: PUT /api/tickets/:id
const updateTicket = async (ticketId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
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

## AI/ML Integration

### AI-Powered Ticket Classification

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import pickle
import os

app = FastAPI()

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    confidence: float
    priority: str

# Load or train model
MODEL_PATH = os.getenv('MODEL_PATH', './models')
vectorizer = TfidfVectorizer(max_features=1000)
classifier = MultinomialNB()

@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    try:
        # Combine title and description
        text = f"{request.title} {request.description}"
        
        # Transform and predict
        text_vector = vectorizer.transform([text])
        category = classifier.predict(text_vector)[0]
        confidence = max(classifier.predict_proba(text_vector)[0])
        
        # Determine priority based on keywords
        priority = determine_priority(text)
        
        return TicketClassificationResponse(
            category=category,
            confidence=float(confidence),
            priority=priority
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def determine_priority(text: str) -> str:
    urgent_keywords = ['urgent', 'critical', 'emergency', 'down', 'broken']
    high_keywords = ['important', 'asap', 'high priority']
    
    text_lower = text.lower()
    if any(word in text_lower for word in urgent_keywords):
        return 'critical'
    elif any(word in text_lower for word in high_keywords):
        return 'high'
    return 'medium'
```

### Risk Detection & Anomaly Analysis

```python
# ml-service/analytics.py
from pydantic import BaseModel
from typing import List
from river import anomaly
from river import preprocessing
import numpy as np

class UserActivity(BaseModel):
    userId: str
    loginCount: int
    tasksCompleted: int
    avgTaskTime: float
    failedLogins: int
    timestamp: str

class RiskAnalysis(BaseModel):
    userId: str
    riskScore: float
    isAnomaly: bool
    factors: List[str]

# Online learning anomaly detector
scaler = preprocessing.StandardScaler()
detector = anomaly.HalfSpaceTrees(n_trees=10, seed=42)

@app.post("/detect-anomaly", response_model=RiskAnalysis)
async def detect_anomaly(activity: UserActivity):
    # Prepare features
    features = {
        'login_count': activity.loginCount,
        'tasks_completed': activity.tasksCompleted,
        'avg_task_time': activity.avgTaskTime,
        'failed_logins': activity.failedLogins
    }
    
    # Scale and detect
    scaled_features = scaler.transform_one(features)
    anomaly_score = detector.score_one(scaled_features)
    detector.learn_one(scaled_features)
    
    # Determine risk factors
    risk_factors = []
    if activity.failedLogins > 3:
        risk_factors.append('Multiple failed login attempts')
    if activity.avgTaskTime > 480:  # 8 hours
        risk_factors.append('Unusually long task completion time')
    if activity.loginCount > 20:
        risk_factors.append('Excessive login frequency')
    
    is_anomaly = anomaly_score > 0.6
    
    return RiskAnalysis(
        userId=activity.userId,
        riskScore=float(anomaly_score),
        isAnomaly=is_anomaly,
        factors=risk_factors
    )
```

### Burnout Detection

```python
# ml-service/burnout.py
from pydantic import BaseModel
from typing import List

class WorkloadData(BaseModel):
    userId: str
    activeTasks: int
    completedThisWeek: int
    avgWorkHours: float
    overtimeHours: float
    missedDeadlines: int

class BurnoutAnalysis(BaseModel):
    userId: str
    burnoutRisk: str  # 'low', 'medium', 'high'
    score: float
    recommendations: List[str]

@app.post("/analyze-burnout", response_model=BurnoutAnalysis)
async def analyze_burnout(workload: WorkloadData):
    # Calculate burnout score (0-100)
    score = 0
    
    # Factor weights
    if workload.activeTasks > 10:
        score += 20
    elif workload.activeTasks > 7:
        score += 10
    
    if workload.avgWorkHours > 50:
        score += 25
    elif workload.avgWorkHours > 45:
        score += 15
    
    if workload.overtimeHours > 10:
        score += 20
    
    if workload.missedDeadlines > 2:
        score += 15
    
    if workload.completedThisWeek < 3:
        score += 10
    
    # Determine risk level
    if score >= 60:
        risk = 'high'
    elif score >= 35:
        risk = 'medium'
    else:
        risk = 'low'
    
    # Generate recommendations
    recommendations = []
    if workload.activeTasks > 8:
        recommendations.append('Redistribute tasks to balance workload')
    if workload.avgWorkHours > 45:
        recommendations.append('Encourage time management and breaks')
    if workload.missedDeadlines > 1:
        recommendations.append('Review deadlines and adjust priorities')
    if workload.overtimeHours > 5:
        recommendations.append('Monitor overtime and enforce work-life balance')
    
    return BurnoutAnalysis(
        userId=workload.userId,
        burnoutRisk=risk,
        score=float(score),
        recommendations=recommendations
    )
```

### Calling ML Service from Backend

```javascript
// backend/services/mlService.js
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL || 'http://localhost:8000';

const classifyTicket = async (title, description) => {
  try {
    const response = await axios.post(`${ML_SERVICE_URL}/classify-ticket`, {
      title,
      description
    });
    return response.data;
  } catch (error) {
    console.error('ML Service Error:', error.message);
    return { category: 'general', confidence: 0, priority: 'medium' };
  }
};

const detectAnomaly = async (userActivity) => {
  try {
    const response = await axios.post(`${ML_SERVICE_URL}/detect-anomaly`, userActivity);
    return response.data;
  } catch (error) {
    console.error('Anomaly Detection Error:', error.message);
    return null;
  }
};

const analyzeBurnout = async (workloadData) => {
  try {
    const response = await axios.post(`${ML_SERVICE_URL}/analyze-burnout`, workloadData);
    return response.data;
  } catch (error) {
    console.error('Burnout Analysis Error:', error.message);
    return null;
  }
};

module.exports = { classifyTicket, detectAnomaly, analyzeBurnout };
```

### Using AI Features in Controllers

```javascript
// backend/controllers/ticketController.js
const { classifyTicket } = require('../services/mlService');
const Ticket = require('../models/Ticket');

const createTicketWithAI = async (req, res) => {
  try {
    const { title, description, createdBy } = req.body;
    
    // Get AI classification
    const aiAnalysis = await classifyTicket(title, description);
    
    // Create ticket with AI suggestions
    const ticket = new Ticket({
      title,
      description,
      createdBy,
      category: aiAnalysis.category,
      priority: aiAnalysis.priority,
      aiConfidence: aiAnalysis.confidence,
      status: 'open'
    });
    
    await ticket.save();
    
    res.status(201).json({
      ticket,
      aiSuggestions: {
        category: aiAnalysis.category,
        priority: aiAnalysis.priority,
        confidence: aiAnalysis.confidence
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

module.exports = { createTicketWithAI };
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
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
    required: true,
    minlength: 6
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
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
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
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  deadline: Date,
  timeTracked: {
    type: Number,
    default: 0  // in seconds
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'hr', 'it-support'],
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
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  aiConfidence: Number,
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### Time Tracking Integration

```javascript
// frontend/src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId, token }) => {
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
    await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ timeTracked: seconds })
    });
  };
  
  const handleStop = () => {
    setIsRunning(false);
    saveTime();
  };
  
  return (
    <div className="time-tracker">
      <div className="time-display">
        {Math.floor(seconds / 3600)}:{Math.floor((seconds % 3600) / 60)}:{seconds % 60}
      </div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      {!isRunning && seconds > 0 && (
        <button onClick={handleStop}>Save</button>
      )}
    </div>
  );
};

export default TimeTracker;
```

### Analytics Dashboard

```javascript
// backend/controllers/analyticsController.js
const User = require('../models/User');
const Task = require('../models/Task');
const Ticket = require('../models/Ticket');

const getDashboardStats = async (req, res) => {
  try {
    const totalUsers = await User.countDocuments({ isActive: true });
    const totalTasks = await Task.countDocuments();
    const completedTasks = await Task.countDocuments({ status: 'done' });
    const openTickets = await Ticket.countDocuments({ status: { $in: ['open', 'in-progress'] } });
    
    // Task distribution by status
    const tasksByStatus = await Task.aggregate([
      { $group: { _id: '$status', count: { $sum: 1 } } }
    ]);
    
    // User performance
    const userPerformance = await Task.aggregate([
      { $match: { status: 'done' } },
      { $group: {
        _id: '$assignedTo',
        tasksCompleted: { $sum: 1 },
        avgCompletionTime: { $avg: '$timeTracked' }
      }},
      { $lookup: {
        from: 'users',
        localField: '_id',
        foreignField: '_id',
        as: 'user'
      }},
      { $unwind: '$user' },
      { $project: {
        userName: '$user.name',
        tasksCompleted: 1,
        avgCompletionTime: 1
      }},
      { $sort: { tasksCompleted: -1 } },
      { $limit: 10 }
    ]);
    
    res.json({
      overview: {
        totalUsers,
        totalTasks,
        completedTasks,
        openTickets,
        completionRate: totalTasks > 0 ? (completedTasks / totalTasks * 100).toFixed(2) : 0
      },
      tasksByStatus,
      topPerformers: userPerformance
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

module.exports = { getDashboardStats };
```

## Troubleshooting

### JWT Token Issues

```javascript
// Check if token is expired
const jwt = require('jsonwebtoken');

const verifyToken = (token) => {
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    return { valid: true, data: decoded };
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      return { valid: false, error: 'Token expired' };
    }
    return { valid: false, error: 'Invalid token' };
  }
};

// Refresh token implementation
const refreshToken = async (oldToken) => {
  try {
    const decoded = jwt.decode(oldToken);
    const newToken = jwt.sign(
      { userId: decoded.userId, role: decoded.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    return newToken;
  } catch (error) {
    throw new Error('Cannot refresh token');
  }
};
```

### ML Service Connection Issues

```javascript
// backend/services/mlService.js with retry logic
const axios = require('axios');

const callMLService = async (endpoint, data, retries = 3) => {
  for (let i = 0; i < retries; i++) {
    try {
      const response = await axios.post(
        `${process.env.ML_SERVICE_URL}${endpoint}`,
        data,
        { timeout: 5000 }
      );
      return response.data;
    } catch (error) {
      if (i === retries - 1) {
        console.error('ML Service unavailable after retries:', error.message);
        return null;
      }
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
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
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000
    });
    console.log('MongoDB Connected');
  } catch (error) {
    console.error('MongoDB Connection Error:', error.message);
    // Retry connection after 5 seconds
    setTimeout(connectDB, 5000);
  }
};

mongoose.connection.on('disconnected', () => {
  console.log('MongoDB Disconnected. Attempting to reconnect...');
  connectDB();
});

module.exports = connectDB;
```

### CORS Configuration

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');

const app = express();

// CORS configuration
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));

app.use(express.json());
```

## Best Practices

### Secure Password Handling

```javascript
// Always use bcrypt with appropriate salt rounds
const bcrypt = require('bcryptjs');

const hashPassword = async (password) => {
  const salt = await bcrypt.genSalt(10);
  return bcrypt.hash(password, salt);
};

// Never log passwords
console.log('User data:', { ...userData, password: '[REDACTED]' });
```

### Rate Limiting

```javascript
// backend/middleware/rateLimiter.js
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 requests per window
  message: 'Too many login attempts, please try again later'
});

const apiLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100 // 100 requests per minute
});

module.exports = { loginLimiter, apiLimiter };
```

### Environment-Based Configuration

```javascript
// backend/config/config.js
module.exports = {
  development: {
    port: 5000,
    mongodb: 'mongodb://localhost:27017/enterprise_dev',
    logLevel: 'debug'
  },
  production: {
    port: process.env.PORT,
    mongodb: process.env.MONGODB_URI,
    logLevel:
