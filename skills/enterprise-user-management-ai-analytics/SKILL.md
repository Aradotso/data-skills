---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management capabilities
triggers:
  - "how do I set up the enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement task tracking with kanban board"
  - "add AI-based ticket classification"
  - "detect user burnout with ML"
  - "build user management dashboard with risk detection"
  - "create support ticket system with AI routing"
  - "implement JWT authentication for user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack application that combines traditional user management with AI-powered analytics. It provides user authentication, task management with Kanban boards, support ticket handling, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and predictive project analytics.

**Architecture:**
- Frontend: React.js
- Backend: Node.js with Express
- ML Service: FastAPI with scikit-learn and River
- Database: MongoDB
- Authentication: JWT

## Installation

### Prerequisites

```bash
# Required software
- Node.js (v14+)
- Python (v3.8+)
- MongoDB (v4.4+)
```

### Clone and Setup

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

**Backend (.env):**
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/user_management
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=24h
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env):**
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

**Frontend (.env):**
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the System

```bash
# Terminal 1 - Backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload
# Runs at http://localhost:8000

# Terminal 3 - Frontend
cd frontend
npm start
# Runs at http://localhost:3000
```

## Backend API Examples

### User Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

const router = express.Router();

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Hash password
    const hashedPassword = await bcrypt.hash(password, 10);
    
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
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
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

// Login
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

### Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { authenticate } = require('../middleware/auth');

const router = express.Router();

// Get user tasks
router.get('/', authenticate, async (req, res) => {
  try {
    const tasks = await Task.find({ 
      assignedTo: req.user.userId 
    }).sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task (Admin only)
router.post('/', authenticate, async (req, res) => {
  try {
    if (req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Access denied' });
    }
    
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      status: 'todo',
      dueDate,
      createdBy: req.user.userId
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', authenticate, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    // Check authorization
    if (task.assignedTo.toString() !== req.user.userId && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Access denied' });
    }
    
    task.status = status;
    task.updatedAt = Date.now();
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Support Ticket System

```javascript
// backend/routes/tickets.js
const express = require('express');
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authenticate } = require('../middleware/auth');

const router = express.Router();

// Create ticket with AI classification
router.post('/', authenticate, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      {
        title,
        description
      }
    );
    
    const { category, urgency, suggestedAssignee } = mlResponse.data;
    
    const ticket = new Ticket({
      title,
      description,
      priority: urgency || priority || 'medium',
      category,
      status: 'open',
      createdBy: req.user.userId,
      assignedTo: suggestedAssignee
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get tickets
router.get('/', authenticate, async (req, res) => {
  try {
    let query = {};
    
    // Regular users see only their tickets
    if (req.user.role === 'user') {
      query.createdBy = req.user.userId;
    }
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise AI Analytics Service")

# Load models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    urgency: str
    suggestedAssignee: Optional[str] = None
    confidence: float

class RiskPredictionRequest(BaseModel):
    userId: str
    recentActivity: List[dict]
    taskCompletionRate: float
    avgResponseTime: float

class RiskPredictionResponse(BaseModel):
    riskLevel: str
    riskScore: float
    factors: List[str]
    recommendations: List[str]

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    workHours: List[float]
    tasksCompleted: int
    tasksAssigned: int
    overtimeHours: float

class BurnoutAnalysisResponse(BaseModel):
    burnoutRisk: str
    score: float
    suggestions: List[str]

@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket using NLP"""
    try:
        # Simple keyword-based classification (replace with actual ML model)
        text = f"{request.title} {request.description}".lower()
        
        # Category detection
        if any(word in text for word in ["bug", "error", "crash", "broken"]):
            category = "technical"
            urgency = "high"
        elif any(word in text for word in ["account", "login", "password", "access"]):
            category = "account"
            urgency = "medium"
        elif any(word in text for word in ["feature", "request", "suggestion"]):
            category = "feature_request"
            urgency = "low"
        else:
            category = "general"
            urgency = "medium"
        
        return TicketClassificationResponse(
            category=category,
            urgency=urgency,
            confidence=0.85
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavior patterns"""
    try:
        risk_score = 0.0
        factors = []
        
        # Analyze completion rate
        if request.taskCompletionRate < 0.5:
            risk_score += 0.3
            factors.append("Low task completion rate")
        
        # Analyze response time
        if request.avgResponseTime > 48:  # hours
            risk_score += 0.25
            factors.append("Slow response time")
        
        # Analyze activity patterns
        if len(request.recentActivity) < 5:
            risk_score += 0.2
            factors.append("Low activity level")
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = "high"
        elif risk_score >= 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        recommendations = []
        if risk_score > 0.5:
            recommendations.append("Schedule one-on-one meeting")
            recommendations.append("Review workload distribution")
            recommendations.append("Provide additional support resources")
        
        return RiskPredictionResponse(
            riskLevel=risk_level,
            riskScore=min(risk_score, 1.0),
            factors=factors,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout", response_model=BurnoutAnalysisResponse)
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze burnout risk based on workload metrics"""
    try:
        score = 0.0
        suggestions = []
        
        # Calculate average work hours
        avg_hours = np.mean(request.workHours) if request.workHours else 0
        
        # Check work hours
        if avg_hours > 50:
            score += 0.4
            suggestions.append("Reduce weekly work hours")
        elif avg_hours > 45:
            score += 0.25
        
        # Check task load
        task_ratio = request.tasksAssigned / max(request.tasksCompleted, 1)
        if task_ratio > 2:
            score += 0.3
            suggestions.append("Redistribute tasks to team members")
        
        # Check overtime
        if request.overtimeHours > 10:
            score += 0.3
            suggestions.append("Limit overtime hours")
        
        # Determine burnout risk
        if score >= 0.7:
            burnout_risk = "high"
        elif score >= 0.4:
            burnout_risk = "moderate"
        else:
            burnout_risk = "low"
        
        if not suggestions:
            suggestions.append("Maintain current work-life balance")
        
        return BurnoutAnalysisResponse(
            burnoutRisk=burnout_risk,
            score=min(score, 1.0),
            suggestions=suggestions
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "timestamp": datetime.now().isoformat()}
```

### Anomaly Detection

```python
# ml-service/anomaly_detection.py
from fastapi import APIRouter
from pydantic import BaseModel
from typing import List
from river import anomaly, preprocessing
import numpy as np

router = APIRouter()

# Initialize online learning model
scaler = preprocessing.StandardScaler()
detector = anomaly.HalfSpaceTrees(seed=42)

class AnomalyRequest(BaseModel):
    userId: str
    loginTime: str
    ipAddress: str
    location: str
    activityCount: int

class AnomalyResponse(BaseModel):
    isAnomaly: bool
    score: float
    severity: str

@router.post("/detect-anomaly", response_model=AnomalyResponse)
async def detect_anomaly(request: AnomalyRequest):
    """Detect anomalous user behavior"""
    try:
        # Extract features
        features = {
            'activity_count': request.activityCount,
            'hour': int(request.loginTime.split(':')[0]) if ':' in request.loginTime else 0
        }
        
        # Scale features
        scaled = scaler.learn_one(features).transform_one(features)
        
        # Get anomaly score
        score = detector.score_one(scaled)
        detector.learn_one(scaled)
        
        # Determine if anomalous
        is_anomaly = score > 0.7
        
        if score > 0.8:
            severity = "critical"
        elif score > 0.6:
            severity = "high"
        elif score > 0.4:
            severity = "medium"
        else:
            severity = "low"
        
        return AnomalyResponse(
            isAnomaly=is_anomaly,
            score=float(score),
            severity=severity
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend React Components

### Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';
import axios from 'axios';

const AuthContext = createContext(null);

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      console.error('Failed to fetch user:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
    delete axios.defaults.headers.common['Authorization'];
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`);
      const taskData = response.data;
      
      setTasks({
        todo: taskData.filter(t => t.status === 'todo'),
        inProgress: taskData.filter(t => t.status === 'inProgress'),
        done: taskData.filter(t => t.status === 'done')
      });
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const onDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const onDrop = (e, status) => {
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, status);
  };

  const onDragOver = (e) => {
    e.preventDefault();
  };

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      <div className="kanban-column" onDrop={(e) => onDrop(e, 'todo')} onDragOver={onDragOver}>
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => (
          <div
            key={task._id}
            className="task-card"
            draggable
            onDragStart={(e) => onDragStart(e, task._id)}
          >
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority ${task.priority}`}>{task.priority}</span>
          </div>
        ))}
      </div>

      <div className="kanban-column" onDrop={(e) => onDrop(e, 'inProgress')} onDragOver={onDragOver}>
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => (
          <div
            key={task._id}
            className="task-card"
            draggable
            onDragStart={(e) => onDragStart(e, task._id)}
          >
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority ${task.priority}`}>{task.priority}</span>
          </div>
        ))}
      </div>

      <div className="kanban-column" onDrop={(e) => onDrop(e, 'done')} onDragOver={onDragOver}>
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => (
          <div
            key={task._id}
            className="task-card completed"
            draggable
            onDragStart={(e) => onDragStart(e, task._id)}
          >
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
// frontend/src/components/AIAnalyticsDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { Line, Bar } from 'react-chartjs-2';
import './AIAnalytics.css';

const AIAnalyticsDashboard = () => {
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const [riskResponse, burnoutResponse] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/api/analytics/risk`),
        axios.get(`${process.env.REACT_APP_API_URL}/api/analytics/burnout`)
      ]);

      setRiskData(riskResponse.data);
      setBurnoutData(burnoutResponse.data);
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  const runRiskPrediction = async (userId) => {
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/predict-risk`,
        {
          userId,
          recentActivity: [],
          taskCompletionRate: 0.75,
          avgResponseTime: 24
        }
      );

      alert(`Risk Level: ${response.data.riskLevel}\nScore: ${response.data.riskScore}`);
    } catch (error) {
      console.error('Risk prediction failed:', error);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI-Powered Analytics</h2>
      
      <div className="analytics-grid">
        <div className="analytics-card">
          <h3>Risk Prediction</h3>
          {riskData && (
            <div>
              <p>High Risk Users: {riskData.highRisk}</p>
              <p>Medium Risk: {riskData.mediumRisk}</p>
              <p>Low Risk: {riskData.lowRisk}</p>
            </div>
          )}
        </div>

        <div className="analytics-card">
          <h3>Burnout Analysis</h3>
          {burnoutData && (
            <div>
              <p>At Risk: {burnoutData.atRisk}</p>
              <p>Average Score: {burnoutData.avgScore.toFixed(2)}</p>
            </div>
          )}
        </div>
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

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
    enum: ['user', 'admin'],
    default: 'user'
  },
  profilePicture: String,
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
  dueDate: Date,
  completedAt: Date,
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
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
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
  category: String,
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
    text: String,
    author: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User'
    },
    createdAt: {
      type: Date,
      default: Date.now
    }
  }],
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authenticate = (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return res.status(401).json({ message: 'Authentication required' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const authorizeAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authenticate, authorizeAdmin };
```

### API Client Setup

```javascript
// frontend/src/utils/apiClient.js
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  timeout: 10000
});

// Request interceptor
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise
