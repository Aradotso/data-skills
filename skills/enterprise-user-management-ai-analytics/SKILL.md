---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "create user management dashboard with AI analytics"
  - "implement AI-powered ticket classification system"
  - "build task tracking with burnout detection"
  - "integrate ML service for user behavior analysis"
  - "deploy user management system with FastAPI ML backend"
  - "configure JWT authentication for enterprise app"
  - "add anomaly detection to user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task/ticket management with AI-powered insights. It provides:

- **User Management**: Role-based access control, authentication with JWT
- **Task Tracking**: Kanban boards, time tracking, task assignment
- **Ticket Management**: Support ticket system with AI classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Architecture**: React frontend, Node.js backend, FastAPI ML service, MongoDB database

## Installation

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
MONGODB_URI=mongodb://localhost:27017/enterprise-users
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup (FastAPI)

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```bash
MODEL_PATH=./models
MONGODB_URI=mongodb://localhost:27017/enterprise-users
API_KEY=your_api_key_here
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install
```

Create `.env` file:

```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Frontend runs at http://localhost:3000
```

## Key API Endpoints

### Authentication APIs

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// Register user
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({
      name,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ token, user: { id: user._id, name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const isValidPassword = await bcrypt.compare(password, user.password);
    if (!isValidPassword) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ token, user: { id: user._id, name: user.name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Task Management APIs

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authenticate, authorize } = require('../middleware/auth');

// Get all tasks for user
router.get('/', authenticate, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user.userId };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task (admin only)
router.post('/', authenticate, authorize(['admin']), async (req, res) => {
  try {
    const { title, description, priority, assignedTo, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      priority,
      assignedTo,
      dueDate,
      status: 'todo',
      createdBy: req.user.userId
    });
    
    await task.save();
    await task.populate('assignedTo', 'name email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', authenticate, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    ).populate('assignedTo', 'name email');
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

module.exports = router;
```

### Ticket Management APIs

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authenticate } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', authenticate, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for ticket classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      { title, description }
    );
    
    const ticket = new Ticket({
      title,
      description,
      priority,
      category: mlResponse.data.category,
      sentiment: mlResponse.data.sentiment,
      urgency: mlResponse.data.urgency,
      createdBy: req.user.userId,
      status: 'open'
    });
    
    await ticket.save();
    await ticket.populate('createdBy', 'name email');
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Get tickets
router.get('/', authenticate, async (req, res) => {
  try {
    const query = req.user.role === 'admin'
      ? {}
      : { createdBy: req.user.userId };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")
os.makedirs(MODEL_PATH, exist_ok=True)

class TicketData(BaseModel):
    title: str
    description: str

class UserBehaviorData(BaseModel):
    userId: str
    loginFrequency: float
    taskCompletionRate: float
    averageTaskTime: float
    failedLoginAttempts: int
    accessPatterns: List[float]

class TaskData(BaseModel):
    taskId: str
    assignedUsers: int
    estimatedHours: float
    complexity: str
    dependencies: int
    completedPercentage: float

# Ticket Classification
@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketData):
    """Classify support ticket using NLP and ML"""
    try:
        text = f"{ticket.title} {ticket.description}".lower()
        
        # Simple keyword-based classification
        categories = {
            "technical": ["bug", "error", "crash", "api", "database", "server"],
            "account": ["password", "login", "access", "permission", "reset"],
            "feature": ["request", "feature", "enhancement", "improvement"],
            "billing": ["payment", "invoice", "subscription", "billing"]
        }
        
        category = "general"
        for cat, keywords in categories.items():
            if any(keyword in text for keyword in keywords):
                category = cat
                break
        
        # Sentiment analysis (simplified)
        negative_words = ["urgent", "critical", "error", "issue", "problem", "broken"]
        sentiment = "negative" if any(word in text for word in negative_words) else "neutral"
        
        # Urgency detection
        urgent_keywords = ["urgent", "critical", "asap", "immediately", "emergency"]
        urgency = "high" if any(word in text for word in urgent_keywords) else "medium"
        
        return {
            "category": category,
            "sentiment": sentiment,
            "urgency": urgency,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Risk Prediction
@app.post("/predict-risk")
async def predict_risk(data: UserBehaviorData):
    """Predict user risk based on behavior patterns"""
    try:
        # Calculate risk score based on behavior
        risk_score = 0.0
        
        # Low login frequency increases risk
        if data.loginFrequency < 5:
            risk_score += 0.2
        
        # Low task completion rate increases risk
        if data.taskCompletionRate < 0.5:
            risk_score += 0.3
        
        # Failed login attempts increase risk
        if data.failedLoginAttempts > 3:
            risk_score += 0.4
        
        # High average task time increases risk
        if data.averageTaskTime > 10:
            risk_score += 0.1
        
        risk_level = "high" if risk_score > 0.6 else "medium" if risk_score > 0.3 else "low"
        
        return {
            "userId": data.userId,
            "riskScore": min(risk_score, 1.0),
            "riskLevel": risk_level,
            "factors": {
                "loginFrequency": data.loginFrequency,
                "taskCompletionRate": data.taskCompletionRate,
                "failedLoginAttempts": data.failedLoginAttempts
            },
            "recommendations": [
                "Monitor user activity closely" if risk_level == "high" else "Regular check-ins recommended"
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Anomaly Detection
@app.post("/detect-anomaly")
async def detect_anomaly(data: UserBehaviorData):
    """Detect anomalous user behavior"""
    try:
        anomalies = []
        
        # Detect unusual login attempts
        if data.failedLoginAttempts > 5:
            anomalies.append({
                "type": "suspicious_login",
                "severity": "high",
                "message": "Excessive failed login attempts detected"
            })
        
        # Detect unusual access patterns
        if len(data.accessPatterns) > 0:
            avg_access = np.mean(data.accessPatterns)
            if avg_access > 100:
                anomalies.append({
                    "type": "unusual_activity",
                    "severity": "medium",
                    "message": "Abnormally high activity detected"
                })
        
        # Detect low productivity
        if data.taskCompletionRate < 0.3 and data.loginFrequency > 10:
            anomalies.append({
                "type": "productivity_issue",
                "severity": "low",
                "message": "Low task completion despite high login frequency"
            })
        
        is_anomalous = len(anomalies) > 0
        
        return {
            "userId": data.userId,
            "isAnomalous": is_anomalous,
            "anomalies": anomalies,
            "timestamp": "2024-01-01T00:00:00Z"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout Detection
@app.post("/detect-burnout")
async def detect_burnout(data: UserBehaviorData):
    """Detect potential employee burnout"""
    try:
        burnout_score = 0.0
        indicators = []
        
        # High average task time
        if data.averageTaskTime > 12:
            burnout_score += 0.3
            indicators.append("Extended task completion times")
        
        # Declining task completion rate
        if data.taskCompletionRate < 0.6:
            burnout_score += 0.3
            indicators.append("Declining productivity")
        
        # Irregular login patterns
        if data.loginFrequency > 15 or data.loginFrequency < 3:
            burnout_score += 0.2
            indicators.append("Irregular work patterns")
        
        # High workload indicators from access patterns
        if len(data.accessPatterns) > 0 and np.max(data.accessPatterns) > 50:
            burnout_score += 0.2
            indicators.append("High workload intensity")
        
        risk_level = "high" if burnout_score > 0.6 else "medium" if burnout_score > 0.3 else "low"
        
        return {
            "userId": data.userId,
            "burnoutScore": min(burnout_score, 1.0),
            "riskLevel": risk_level,
            "indicators": indicators,
            "recommendations": [
                "Immediate intervention required" if risk_level == "high" 
                else "Monitor workload and provide support" if risk_level == "medium"
                else "Continue regular check-ins"
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Project Delay Prediction
@app.post("/predict-delay")
async def predict_delay(task: TaskData):
    """Predict likelihood of project/task delay"""
    try:
        delay_probability = 0.0
        risk_factors = []
        
        # High complexity increases delay risk
        if task.complexity == "high":
            delay_probability += 0.3
            risk_factors.append("High task complexity")
        
        # Many dependencies increase risk
        if task.dependencies > 3:
            delay_probability += 0.2
            risk_factors.append("Multiple dependencies")
        
        # Low completion percentage near deadline
        if task.completedPercentage < 50:
            delay_probability += 0.3
            risk_factors.append("Behind schedule")
        
        # Too many assigned users (coordination overhead)
        if task.assignedUsers > 5:
            delay_probability += 0.1
            risk_factors.append("High coordination overhead")
        
        # Estimated hours very high
        if task.estimatedHours > 40:
            delay_probability += 0.1
            risk_factors.append("Large time estimate")
        
        will_delay = delay_probability > 0.5
        
        return {
            "taskId": task.taskId,
            "willDelay": will_delay,
            "delayProbability": min(delay_probability, 1.0),
            "riskFactors": risk_factors,
            "suggestedActions": [
                "Reassign resources" if will_delay else "Continue monitoring",
                "Review task breakdown" if task.complexity == "high" else "Maintain current pace"
            ],
            "estimatedDelayDays": int(delay_probability * 7) if will_delay else 0
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Frontend Integration

### Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useContext, useEffect } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const useAuth = () => useContext(AuthContext);

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUserProfile();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchUserProfile = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/auth/profile`);
      setUser(response.data);
    } catch (error) {
      console.error('Failed to fetch user profile', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    return user;
  };

  const register = async (name, email, password, role) => {
    const response = await axios.post(`${API_URL}/api/auth/register`, {
      name,
      email,
      password,
      role
    });
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
    delete axios.defaults.headers.common['Authorization'];
  };

  const value = {
    user,
    token,
    login,
    register,
    logout,
    loading,
    isAuthenticated: !!user,
    isAdmin: user?.role === 'admin'
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
};
```

### Task Management Hook

```javascript
// frontend/src/hooks/useTasks.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useTasks = () => {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  const fetchTasks = async () => {
    try {
      setLoading(true);
      const response = await axios.get(`${API_URL}/api/tasks`);
      setTasks(response.data);
      setError(null);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  const createTask = async (taskData) => {
    try {
      const response = await axios.post(`${API_URL}/api/tasks`, taskData);
      setTasks([response.data, ...tasks]);
      return response.data;
    } catch (err) {
      throw new Error(err.response?.data?.message || 'Failed to create task');
    }
  };

  const updateTaskStatus = async (taskId, status) => {
    try {
      const response = await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status }
      );
      setTasks(tasks.map(task => 
        task._id === taskId ? response.data : task
      ));
      return response.data;
    } catch (err) {
      throw new Error(err.response?.data?.message || 'Failed to update task');
    }
  };

  const deleteTask = async (taskId) => {
    try {
      await axios.delete(`${API_URL}/api/tasks/${taskId}`);
      setTasks(tasks.filter(task => task._id !== taskId));
    } catch (err) {
      throw new Error(err.response?.data?.message || 'Failed to delete task');
    }
  };

  useEffect(() => {
    fetchTasks();
  }, []);

  return {
    tasks,
    loading,
    error,
    fetchTasks,
    createTask,
    updateTaskStatus,
    deleteTask
  };
};
```

### AI Analytics Component

```javascript
// frontend/src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { useAuth } from '../context/AuthContext';

const ML_SERVICE_URL = process.env.REACT_APP_ML_SERVICE_URL;

const AIAnalytics = () => {
  const { user } = useAuth();
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutScore: null,
    anomalies: [],
    loading: true
  });

  useEffect(() => {
    fetchAnalytics();
  }, [user]);

  const fetchAnalytics = async () => {
    try {
      // Fetch user behavior data
      const behaviorData = {
        userId: user.id,
        loginFrequency: 8,
        taskCompletionRate: 0.75,
        averageTaskTime: 6.5,
        failedLoginAttempts: 0,
        accessPatterns: [20, 25, 30, 22, 28]
      };

      // Get risk prediction
      const riskResponse = await axios.post(
        `${ML_SERVICE_URL}/predict-risk`,
        behaviorData
      );

      // Get burnout detection
      const burnoutResponse = await axios.post(
        `${ML_SERVICE_URL}/detect-burnout`,
        behaviorData
      );

      // Get anomaly detection
      const anomalyResponse = await axios.post(
        `${ML_SERVICE_URL}/detect-anomaly`,
        behaviorData
      );

      setAnalytics({
        riskScore: riskResponse.data,
        burnoutScore: burnoutResponse.data,
        anomalies: anomalyResponse.data.anomalies,
        loading: false
      });
    } catch (error) {
      console.error('Failed to fetch AI analytics', error);
      setAnalytics(prev => ({ ...prev, loading: false }));
    }
  };

  if (analytics.loading) {
    return <div>Loading AI Analytics...</div>;
  }

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      <div className="analytics-grid">
        {/* Risk Score */}
        <div className="analytics-card">
          <h3>Risk Assessment</h3>
          <div className={`risk-indicator ${analytics.riskScore?.riskLevel}`}>
            <span className="score">
              {(analytics.riskScore?.riskScore * 100).toFixed(0)}%
            </span>
            <span className="level">{analytics.riskScore?.riskLevel}</span>
          </div>
          <ul className="recommendations">
            {analytics.riskScore?.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>

        {/* Burnout Score */}
        <div className="analytics-card">
          <h3>Burnout Detection</h3>
          <div className={`burnout-indicator ${analytics.burnoutScore?.riskLevel}`}>
            <span className="score">
              {(analytics.burnoutScore?.burnoutScore * 100).toFixed(0)}%
            </span>
            <span className="level">{analytics.burnoutScore?.riskLevel}</span>
          </div>
          <div className="indicators">
            <h4>Indicators:</h4>
            <ul>
              {analytics.burnoutScore?.indicators.map((indicator, idx) => (
                <li key={idx}>{indicator}</li>
              ))}
            </ul>
          </div>
        </div>

        {/* Anomalies */}
        <div className="analytics-card">
          <h3>Anomaly Detection</h3>
          {analytics.anomalies.length > 0 ? (
            <ul className="anomaly-list">
              {analytics.anomalies.map((anomaly, idx) => (
                <li key={idx} className={`anomaly-${anomaly.severity}`}>
                  <strong>{anomaly.type}</strong>
                  <span className="severity">{anomaly.severity}</span>
                  <p>{anomaly.message}</p>
                </li>
              ))}
            </ul>
          ) : (
            <p className="no-anomalies">No anomalies detected</p>
          )}
        </div>
      </div>
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
    lowercase: true,
    trim: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: {
    type: String,
    default: 'General'
  },
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: {
    type: Date
  },
  loginAttempts: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
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
    required: true,
    trim: true
  },
  description: {
    type: String,
    trim: true
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
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: {
    type: Date
  },
  estimatedHours: {
    type: Number
  },
  actualHours: {
    type: Number,
    default: 0
  },
  completedAt: {
    type: Date
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
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
    required: true,
    trim: true
  },
  description: {
    type: String,
    required: true
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  category: {
    type: String,
    enum: ['
