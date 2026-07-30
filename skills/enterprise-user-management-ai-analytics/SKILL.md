---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and task management
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement task tracking with AI insights"
  - "create admin dashboard with user analytics"
  - "add AI-based ticket classification"
  - "build burnout detection system"
  - "deploy user management with anomaly detection"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system combining React frontend, Node.js backend, and FastAPI ML service. Provides intelligent user management, task tracking, support ticketing, and AI-powered analytics including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

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

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
```

### ML Service Setup (FastAPI)

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install
```

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Project Architecture

The system consists of three main components:

1. **Frontend (React)**: User interface for admins and users
2. **Backend (Node.js)**: REST API handling authentication, user management, tasks, and tickets
3. **ML Service (FastAPI)**: AI models for analytics and predictions

## Backend API Structure

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
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
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

### User Management Endpoints

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const auth = require('../middleware/auth');
const adminAuth = require('../middleware/adminAuth');

// Get all users (Admin only)
router.get('/', [auth, adminAuth], async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user by ID
router.get('/:id', auth, async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json(user);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update user (Admin only)
router.put('/:id', [auth, adminAuth], async (req, res) => {
  try {
    const { username, email, role, status } = req.body;
    
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { username, email, role, status },
      { new: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Delete user (Admin only)
router.delete('/:id', [auth, adminAuth], async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: error.message });
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
const auth = require('../middleware/auth');

// Create task
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      createdBy: req.user.userId,
      priority: priority || 'medium',
      dueDate,
      status: 'todo'
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get tasks for user
router.get('/my-tasks', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Track time for task
router.post('/:id/time-log', auth, async (req, res) => {
  try {
    const { duration } = req.body;
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.timeTracking.push({
      userId: req.user.userId,
      duration,
      timestamp: Date.now()
    });
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Ticket Management Endpoints

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const auth = require('../middleware/auth');
const axios = require('axios');

// Create ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
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
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', auth, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.userId })
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI Application Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from typing import List, Dict
import pickle
import os
from datetime import datetime

app = FastAPI(title="Enterprise User Management ML Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    lastLoginTime: str
    accessLocations: List[str]

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    tasksCompleted: int
    avgTaskDuration: float
    workingHours: float
    missedDeadlines: int

class TicketClassificationResponse(BaseModel):
    category: str
    priority: str
    confidence: float

class RiskPredictionResponse(BaseModel):
    riskScore: float
    riskLevel: str
    factors: List[str]

@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    """
    Classify support ticket using NLP
    """
    try:
        # Simple rule-based classification (replace with trained model)
        text = f"{request.title} {request.description}".lower()
        
        # Category classification
        if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
            priority = 'high'
        elif any(word in text for word in ['password', 'login', 'access']):
            category = 'account'
            priority = 'medium'
        elif any(word in text for word in ['feature', 'enhancement', 'suggestion']):
            category = 'feature_request'
            priority = 'low'
        else:
            category = 'general'
            priority = 'medium'
        
        return TicketClassificationResponse(
            category=category,
            priority=priority,
            confidence=0.85
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    """
    Predict user security risk based on behavior
    """
    try:
        risk_score = 0.0
        factors = []
        
        # Failed login analysis
        if request.failedLogins > 3:
            risk_score += 0.3
            factors.append("Multiple failed login attempts")
        
        # Login attempts analysis
        if request.loginAttempts > 10:
            risk_score += 0.2
            factors.append("Unusual number of login attempts")
        
        # Location analysis
        if len(request.accessLocations) > 3:
            risk_score += 0.25
            factors.append("Multiple access locations")
        
        # Time-based analysis
        try:
            last_login = datetime.fromisoformat(request.lastLoginTime)
            hours_since_login = (datetime.now() - last_login).total_seconds() / 3600
            if hours_since_login < 0.5 and request.loginAttempts > 5:
                risk_score += 0.25
                factors.append("Rapid successive login attempts")
        except:
            pass
        
        # Normalize risk score
        risk_score = min(risk_score, 1.0)
        
        # Determine risk level
        if risk_score < 0.3:
            risk_level = "low"
        elif risk_score < 0.6:
            risk_level = "medium"
        else:
            risk_level = "high"
        
        return RiskPredictionResponse(
            riskScore=round(risk_score, 2),
            riskLevel=risk_level,
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """
    Analyze user burnout risk based on workload metrics
    """
    try:
        burnout_score = 0.0
        indicators = []
        
        # Working hours analysis
        if request.workingHours > 50:
            burnout_score += 0.35
            indicators.append("Excessive working hours")
        
        # Task completion rate
        if request.avgTaskDuration > 8:
            burnout_score += 0.25
            indicators.append("Prolonged task completion time")
        
        # Missed deadlines
        if request.missedDeadlines > 2:
            burnout_score += 0.3
            indicators.append("Frequent missed deadlines")
        
        # Task overload
        if request.tasksCompleted > 20 and request.avgTaskDuration > 6:
            burnout_score += 0.1
            indicators.append("High task volume with long duration")
        
        burnout_score = min(burnout_score, 1.0)
        
        if burnout_score < 0.3:
            risk_level = "low"
            recommendation = "Employee workload is healthy"
        elif burnout_score < 0.6:
            risk_level = "moderate"
            recommendation = "Consider workload redistribution"
        else:
            risk_level = "high"
            recommendation = "Immediate intervention required"
        
        return {
            "userId": request.userId,
            "burnoutScore": round(burnout_score, 2),
            "riskLevel": risk_level,
            "indicators": indicators,
            "recommendation": recommendation
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/anomaly-detection")
async def detect_anomaly(data: Dict):
    """
    Detect anomalies in user behavior
    """
    try:
        anomalies = []
        
        # Check for unusual patterns
        if data.get("loginTime"):
            hour = int(data["loginTime"].split(":")[0])
            if hour < 6 or hour > 22:
                anomalies.append({
                    "type": "unusual_login_time",
                    "severity": "medium",
                    "description": "Login outside normal business hours"
                })
        
        if data.get("dataAccess", 0) > 100:
            anomalies.append({
                "type": "excessive_data_access",
                "severity": "high",
                "description": "Unusually high data access volume"
            })
        
        return {
            "isAnomaly": len(anomalies) > 0,
            "anomalyCount": len(anomalies),
            "anomalies": anomalies
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Frontend Integration

### API Service Setup

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

// Create axios instance
const api = axios.create({
  baseURL: API_URL,
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authAPI = {
  login: (credentials) => api.post('/auth/login', credentials),
  register: (userData) => api.post('/auth/register', userData),
};

export const userAPI = {
  getAll: () => api.get('/users'),
  getById: (id) => api.get(`/users/${id}`),
  update: (id, data) => api.put(`/users/${id}`, data),
  delete: (id) => api.delete(`/users/${id}`),
};

export const taskAPI = {
  create: (task) => api.post('/tasks', task),
  getMyTasks: () => api.get('/tasks/my-tasks'),
  updateStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  logTime: (id, duration) => api.post(`/tasks/${id}/time-log`, { duration }),
};

export const ticketAPI = {
  create: (ticket) => api.post('/tickets', ticket),
  getMyTickets: () => api.get('/tickets/my-tickets'),
};

export const mlAPI = {
  predictRisk: (data) => axios.post(`${ML_API_URL}/predict-risk`, data),
  analyzeBurnout: (data) => axios.post(`${ML_API_URL}/analyze-burnout`, data),
  detectAnomaly: (data) => axios.post(`${ML_API_URL}/anomaly-detection`, data),
};

export default api;
```

### User Dashboard Component

```javascript
// frontend/src/components/UserDashboard.js
import React, { useState, useEffect } from 'react';
import { taskAPI, mlAPI } from '../services/api';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [burnoutAnalysis, setBurnoutAnalysis] = useState(null);
  const [activeTimer, setActiveTimer] = useState(null);
  const [elapsedTime, setElapsedTime] = useState(0);

  useEffect(() => {
    fetchTasks();
    fetchBurnoutAnalysis();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskAPI.getMyTasks();
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done'),
      };
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const fetchBurnoutAnalysis = async () => {
    try {
      const userId = localStorage.getItem('userId');
      const response = await mlAPI.analyzeBurnout({
        userId,
        tasksCompleted: 15,
        avgTaskDuration: 6.5,
        workingHours: 45,
        missedDeadlines: 1,
      });
      setBurnoutAnalysis(response.data);
    } catch (error) {
      console.error('Error fetching burnout analysis:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await taskAPI.updateStatus(taskId, newStatus);
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const startTimer = (taskId) => {
    setActiveTimer(taskId);
    setElapsedTime(0);
  };

  const stopTimer = async (taskId) => {
    try {
      await taskAPI.logTime(taskId, elapsedTime);
      setActiveTimer(null);
      setElapsedTime(0);
    } catch (error) {
      console.error('Error logging time:', error);
    }
  };

  useEffect(() => {
    let interval;
    if (activeTimer) {
      interval = setInterval(() => {
        setElapsedTime(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [activeTimer]);

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {burnoutAnalysis && (
        <div className={`burnout-alert ${burnoutAnalysis.riskLevel}`}>
          <h3>Burnout Analysis</h3>
          <p>Risk Level: {burnoutAnalysis.riskLevel}</p>
          <p>Score: {burnoutAnalysis.burnoutScore}</p>
          <p>{burnoutAnalysis.recommendation}</p>
        </div>
      )}

      <div className="kanban-board">
        <div className="column">
          <h2>To Do</h2>
          {tasks.todo.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => moveTask(task._id, 'in_progress')}>
                Start
              </button>
            </div>
          ))}
        </div>

        <div className="column">
          <h2>In Progress</h2>
          {tasks.inProgress.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              {activeTimer === task._id ? (
                <div>
                  <p>Time: {formatTime(elapsedTime)}</p>
                  <button onClick={() => stopTimer(task._id)}>Stop</button>
                </div>
              ) : (
                <button onClick={() => startTimer(task._id)}>Start Timer</button>
              )}
              <button onClick={() => moveTask(task._id, 'done')}>
                Complete
              </button>
            </div>
          ))}
        </div>

        <div className="column">
          <h2>Done</h2>
          {tasks.done.map(task => (
            <div key={task._id} className="task-card completed">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className="badge">✓ Completed</span>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component

```javascript
// frontend/src/components/AdminAnalytics.js
import React, { useState, useEffect } from 'react';
import { userAPI, mlAPI } from '../services/api';

const AdminAnalytics = () => {
  const [users, setUsers] = useState([]);
  const [riskAnalysis, setRiskAnalysis] = useState([]);

  useEffect(() => {
    fetchUsers();
  }, []);

  const fetchUsers = async () => {
    try {
      const response = await userAPI.getAll();
      setUsers(response.data);
      analyzeUserRisks(response.data);
    } catch (error) {
      console.error('Error fetching users:', error);
    }
  };

  const analyzeUserRisks = async (userList) => {
    try {
      const analyses = await Promise.all(
        userList.map(async (user) => {
          const riskData = await mlAPI.predictRisk({
            userId: user._id,
            loginAttempts: user.loginAttempts || 0,
            failedLogins: user.failedLogins || 0,
            lastLoginTime: user.lastLogin || new Date().toISOString(),
            accessLocations: user.accessLocations || [],
          });
          return { userId: user._id, username: user.username, ...riskData.data };
        })
      );
      setRiskAnalysis(analyses);
    } catch (error) {
      console.error('Error analyzing risks:', error);
    }
  };

  return (
    <div className="admin-analytics">
      <h1>Security & Risk Analytics</h1>
      
      <div className="risk-overview">
        <h2>User Risk Assessment</h2>
        <table>
          <thead>
            <tr>
              <th>User</th>
              <th>Risk Score</th>
              <th>Risk Level</th>
              <th>Factors</th>
              <th>Action</th>
            </tr>
          </thead>
          <tbody>
            {riskAnalysis.map((analysis) => (
              <tr key={analysis.userId} className={`risk-${analysis.riskLevel}`}>
                <td>{analysis.username}</td>
                <td>{analysis.riskScore}</td>
                <td>
                  <span className={`badge ${analysis.riskLevel}`}>
                    {analysis.riskLevel}
                  </span>
                </td>
                <td>
                  <ul>
                    {analysis.factors.map((factor, idx) => (
                      <li key={idx}>{factor}</li>
                    ))}
                  </ul>
                </td>
                <td>
                  {analysis.riskLevel === 'high' && (
                    <button className="btn-danger">Investigate</button>
                  )}
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AdminAnalytics;
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
    unique: true,
  },
  email: {
    type: String,
    required: true,
    unique: true,
  },
  password: {
    type: String,
    required: true,
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user',
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active',
  },
  loginAttempts: {
    type: Number,
    default: 0,
  },
  failedLogins: {
    type: Number,
    default: 0,
  },
  lastLogin: {
    type: Date,
  },
  accessLocations: [String],
  createdAt: {
    type: Date,
    default: Date.now,
  },
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
  },
  description: {
    type: String,
  },
  status: {
    type: String,
    enum: ['todo', 'in_progress', 'done'],
    default: 'todo',
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium',
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User
