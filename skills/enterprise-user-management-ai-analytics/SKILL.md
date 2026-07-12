---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task tracking
triggers:
  - "set up enterprise user management with AI analytics"
  - "create a user management dashboard with task tracking"
  - "implement AI-based ticket classification and risk detection"
  - "build a kanban board with time tracking for users"
  - "add anomaly detection and burnout analysis to user system"
  - "integrate ML service for predictive project insights"
  - "configure JWT authentication for user management app"
  - "deploy user management system with FastAPI ML backend"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise application for managing users, tasks, and support tickets with integrated AI/ML capabilities including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: CRUD operations with role-based access control (Admin/User)
- **Task Management**: Kanban board with drag-and-drop, time tracking, and progress monitoring
- **Support Tickets**: AI-powered classification, routing, and priority assignment
- **AI Analytics**: Risk detection, anomaly detection, burnout prediction, and project delay forecasting
- **Authentication**: JWT-based secure authentication and authorization
- **Audit Logging**: Track all system activities and user actions

## Architecture Overview

The system consists of three main components:

1. **Frontend** (React.js) - Port 3000
2. **Backend** (Node.js) - Port 5000
3. **ML Service** (FastAPI + scikit-learn) - Port 8000

## Installation

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup (Node.js)

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/user-management
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup (FastAPI)

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=info
ENABLE_ONLINE_LEARNING=true
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Backend API Reference

### Authentication Endpoints

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
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
      name,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate token
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email }).select('+password');
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Task Management Endpoints

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { protect, authorize } = require('../middleware/auth');

// Get all tasks (Admin) or user's tasks
router.get('/', protect, async (req, res) => {
  try {
    const filter = req.user.role === 'admin' ? {} : { assignedTo: req.user.id };
    const tasks = await Task.find(filter)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, count: tasks.length, data: tasks });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Create task (Admin only)
router.post('/', protect, authorize('admin'), async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      status: 'todo',
      dueDate,
      createdBy: req.user.id
    });
    
    await task.save();
    await task.populate('assignedTo', 'name email');
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    // Check authorization
    if (req.user.role !== 'admin' && task.assignedTo.toString() !== req.user.id) {
      return res.status(403).json({ message: 'Not authorized' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Support Ticket Endpoints with AI Integration

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { protect } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, category } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
      title,
      description
    });
    
    const { priority, suggested_category, confidence } = mlResponse.data;
    
    const ticket = new Ticket({
      title,
      description,
      category: category || suggested_category,
      priority,
      aiConfidence: confidence,
      createdBy: req.user.id,
      status: 'open'
    });
    
    await ticket.save();
    
    res.status(201).json({
      success: true,
      data: ticket,
      aiInsights: {
        suggestedCategory: suggested_category,
        confidence: confidence
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Get tickets with filtering
router.get('/', protect, async (req, res) => {
  try {
    const { status, priority, category } = req.query;
    
    const filter = req.user.role === 'admin' ? {} : { createdBy: req.user.id };
    
    if (status) filter.status = status;
    if (priority) filter.priority = priority;
    if (category) filter.category = category;
    
    const tickets = await Ticket.find(filter)
      .populate('createdBy', 'name email')
      .populate('assignedTo', 'name email')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, count: tickets.length, data: tickets });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

## ML Service API

### FastAPI ML Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Dict, Optional
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, metrics
import joblib
import os

app = FastAPI(title="User Management ML Service")

# Models
class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_frequency: float
    failed_logins: int
    unusual_hours: int
    data_access_volume: float

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_pending: int
    avg_task_duration: float
    overtime_hours: float
    days_since_break: int

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Ticket classifier
try:
    ticket_vectorizer = joblib.load(f'{MODEL_PATH}/ticket_vectorizer.pkl')
    ticket_classifier = joblib.load(f'{MODEL_PATH}/ticket_classifier.pkl')
except:
    ticket_vectorizer = TfidfVectorizer(max_features=100)
    ticket_classifier = RandomForestClassifier(n_estimators=50)

# Anomaly detector (online learning)
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, seed=42)

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket and predict priority"""
    try:
        text = f"{request.title} {request.description}"
        
        # Simple rule-based classification for demo
        text_lower = text.lower()
        
        # Priority detection
        if any(word in text_lower for word in ['urgent', 'critical', 'emergency', 'asap']):
            priority = 'high'
            confidence = 0.9
        elif any(word in text_lower for word in ['important', 'soon', 'needed']):
            priority = 'medium'
            confidence = 0.75
        else:
            priority = 'low'
            confidence = 0.6
        
        # Category detection
        if any(word in text_lower for word in ['login', 'password', 'access', 'authentication']):
            category = 'authentication'
        elif any(word in text_lower for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
        elif any(word in text_lower for word in ['feature', 'request', 'enhancement']):
            category = 'feature_request'
        else:
            category = 'general'
        
        return {
            "priority": priority,
            "suggested_category": category,
            "confidence": confidence
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavior patterns"""
    try:
        # Calculate risk score
        risk_score = 0
        factors = []
        
        # Failed logins
        if request.failed_logins > 5:
            risk_score += 30
            factors.append("High failed login attempts")
        elif request.failed_logins > 2:
            risk_score += 15
        
        # Unusual hours
        if request.unusual_hours > 10:
            risk_score += 25
            factors.append("Frequent unusual hour access")
        elif request.unusual_hours > 5:
            risk_score += 10
        
        # Data access volume
        if request.data_access_volume > 1000:
            risk_score += 20
            factors.append("Abnormal data access volume")
        elif request.data_access_volume > 500:
            risk_score += 10
        
        # Login frequency anomaly
        if request.login_frequency > 50:
            risk_score += 15
            factors.append("Unusually high login frequency")
        
        risk_level = "high" if risk_score >= 50 else "medium" if risk_score >= 25 else "low"
        
        return {
            "user_id": request.user_id,
            "risk_score": min(risk_score, 100),
            "risk_level": risk_level,
            "risk_factors": factors,
            "recommendation": "Immediate review required" if risk_level == "high" else "Monitor closely" if risk_level == "medium" else "Normal activity"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(data: Dict):
    """Detect anomalies in user behavior using online learning"""
    try:
        features = np.array([
            data.get('login_count', 0),
            data.get('task_completion_rate', 0),
            data.get('avg_session_duration', 0),
            data.get('failed_attempts', 0)
        ])
        
        # Score the observation
        score = anomaly_detector.score_one(features)
        
        # Update model
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.6
        
        return {
            "is_anomaly": is_anomaly,
            "anomaly_score": float(score),
            "severity": "high" if score > 0.8 else "medium" if score > 0.6 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze user burnout risk"""
    try:
        burnout_score = 0
        indicators = []
        
        # High pending tasks
        if request.tasks_pending > 20:
            burnout_score += 25
            indicators.append("High pending task load")
        elif request.tasks_pending > 10:
            burnout_score += 15
        
        # Long task durations
        if request.avg_task_duration > 8:
            burnout_score += 20
            indicators.append("Tasks taking longer than expected")
        
        # Overtime
        if request.overtime_hours > 15:
            burnout_score += 30
            indicators.append("Excessive overtime hours")
        elif request.overtime_hours > 5:
            burnout_score += 15
        
        # No breaks
        if request.days_since_break > 30:
            burnout_score += 25
            indicators.append("No recent time off")
        elif request.days_since_break > 14:
            burnout_score += 10
        
        burnout_level = "high" if burnout_score >= 60 else "medium" if burnout_score >= 30 else "low"
        
        return {
            "user_id": request.user_id,
            "burnout_score": min(burnout_score, 100),
            "burnout_level": burnout_level,
            "indicators": indicators,
            "recommendation": "Immediate intervention needed" if burnout_level == "high" else "Consider workload adjustment" if burnout_level == "medium" else "Healthy workload"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Frontend Integration

### React Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  const API_URL = process.env.REACT_APP_API_URL;

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
      const response = await axios.get(`${API_URL}/auth/me`);
      setUser(response.data.data);
    } catch (error) {
      console.error('Auth error:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/auth/login`, { email, password });
    const { token, user } = response.data;
    
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    
    return user;
  };

  const register = async (userData) => {
    const response = await axios.post(`${API_URL}/auth/register`, userData);
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

  return (
    <AuthContext.Provider value={{ user, login, register, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const [loading, setLoading] = useState(true);
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/tasks`);
      const taskData = response.data.data;
      
      const grouped = {
        todo: taskData.filter(t => t.status === 'todo'),
        inprogress: taskData.filter(t => t.status === 'inprogress'),
        done: taskData.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/tasks/${taskId}/status`, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = (e, status) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, status);
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {['todo', 'inprogress', 'done'].map(status => (
        <div 
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3 className="column-title">
            {status === 'todo' ? 'To Do' : status === 'inprogress' ? 'In Progress' : 'Done'}
            <span className="task-count">{tasks[status].length}</span>
          </h3>
          
          <div className="task-list">
            {tasks[status].map(task => (
              <div
                key={task._id}
                className={`task-card priority-${task.priority}`}
                draggable
                onDragStart={(e) => handleDragStart(e, task._id)}
              >
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <div className="task-meta">
                  <span className={`priority-badge ${task.priority}`}>
                    {task.priority}
                  </span>
                  {task.dueDate && (
                    <span className="due-date">
                      Due: {new Date(task.dueDate).toLocaleDateString()}
                    </span>
                  )}
                </div>
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

### AI Risk Dashboard Component

```javascript
// frontend/src/components/AIRiskDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './AIRiskDashboard.css';

const AIRiskDashboard = () => {
  const [riskData, setRiskData] = useState([]);
  const [loading, setLoading] = useState(true);
  const ML_URL = process.env.REACT_APP_ML_URL;
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchRiskAnalysis();
  }, []);

  const fetchRiskAnalysis = async () => {
    try {
      // Fetch users and their activity data
      const usersResponse = await axios.get(`${API_URL}/users/activity`);
      const users = usersResponse.data.data;
      
      // Analyze each user
      const riskPromises = users.map(async (user) => {
        const riskResponse = await axios.post(`${ML_URL}/predict-risk`, {
          user_id: user._id,
          login_frequency: user.loginFrequency || 0,
          failed_logins: user.failedLogins || 0,
          unusual_hours: user.unusualHours || 0,
          data_access_volume: user.dataAccessVolume || 0
        });
        
        return {
          user: user.name,
          email: user.email,
          ...riskResponse.data
        };
      });
      
      const risks = await Promise.all(riskPromises);
      setRiskData(risks.sort((a, b) => b.risk_score - a.risk_score));
    } catch (error) {
      console.error('Error fetching risk analysis:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Analyzing risks...</div>;

  return (
    <div className="ai-risk-dashboard">
      <h2>AI Risk Analysis</h2>
      
      <div className="risk-summary">
        <div className="risk-stat high">
          <h3>{riskData.filter(r => r.risk_level === 'high').length}</h3>
          <p>High Risk Users</p>
        </div>
        <div className="risk-stat medium">
          <h3>{riskData.filter(r => r.risk_level === 'medium').length}</h3>
          <p>Medium Risk Users</p>
        </div>
        <div className="risk-stat low">
          <h3>{riskData.filter(r => r.risk_level === 'low').length}</h3>
          <p>Low Risk Users</p>
        </div>
      </div>

      <div className="risk-table">
        <table>
          <thead>
            <tr>
              <th>User</th>
              <th>Risk Score</th>
              <th>Risk Level</th>
              <th>Risk Factors</th>
              <th>Recommendation</th>
            </tr>
          </thead>
          <tbody>
            {riskData.map((risk, index) => (
              <tr key={index} className={`risk-${risk.risk_level}`}>
                <td>
                  <div>{risk.user}</div>
                  <small>{risk.email}</small>
                </td>
                <td>
                  <div className="risk-score">{risk.risk_score}</div>
                </td>
                <td>
                  <span className={`badge ${risk.risk_level}`}>
                    {risk.risk_level.toUpperCase()}
                  </span>
                </td>
                <td>
                  <ul className="risk-factors">
                    {risk.risk_factors.map((factor, i) => (
                      <li key={i}>{factor}</li>
                    ))}
                  </ul>
                </td>
                <td>{risk.recommendation}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AIRiskDashboard;
```

## Database Models

### MongoDB User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please provide a name'],
    trim: true
  },
  email: {
    type: String,
    required: [true, 'Please provide an email'],
    unique: true,
    lowercase: true,
    match: [/^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/, 'Please provide a valid email']
  },
  password: {
    type: String,
    required: [true, 'Please provide a password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  position: String,
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: Date,
  loginFrequency: {
    type: Number,
    default: 0
  },
  failedLogins: {
    type: Number,
    default: 0
  },
  unusualHours: {
    type: Number,
    default: 0
  },
  dataAccessVolume: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('User', UserSchema);
```

### MongoDB Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: [true, 'Please provide a task title'],
    trim: true
  },
  description: {
    type: String,
    required: [true, 'Please provide a task description']
  },
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
  timeTracked: {
    type: Number,
    default: 0  // in minutes
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt:
