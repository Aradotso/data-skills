---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, ticket routing, and risk detection
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with ML"
  - "configure ticket classification system"
  - "add burnout detection to user system"
  - "build admin dashboard with AI insights"
  - "integrate risk prediction for users"
  - "develop task management with anomaly detection"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines user/task management with AI-powered analytics. It provides role-based access control, Kanban boards, ticket management, and ML-driven insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

**Stack:** React.js frontend, Node.js backend, FastAPI ML service, MongoDB database, JWT authentication.

## Installation

### Prerequisites
```bash
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
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
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

Create `ml-service/.env`:
```env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=info
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
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
# Frontend runs at http://localhost:3000
```

## Backend API Patterns

### Authentication
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
      return res.status(400).json({ error: 'User already exists' });
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
      { expiresIn: '24h' }
    );
    
    res.status(201).json({ token, user: { id: user._id, name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const isValidPassword = await bcrypt.compare(password, user.password);
    if (!isValidPassword) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({ token, user: { id: user._id, name: user.name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Authentication Middleware
```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token' });
    }
    
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

### User Management Routes
```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { authMiddleware, adminOnly } = require('../middleware/auth');

// Get all users (admin only)
router.get('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user profile
router.get('/profile', authMiddleware, async (req, res) => {
  try {
    const user = await User.findById(req.user.userId).select('-password');
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update user (admin only)
router.put('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const { name, email, role, status } = req.body;
    
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { name, email, role, status, updatedAt: Date.now() },
      { new: true }
    ).select('-password');
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Delete user (admin only)
router.delete('/:id', authMiddleware, adminOnly, async (req, res) => {
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
const { authMiddleware } = require('../middleware/auth');

// Get user tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      status: 'todo',
      createdBy: req.user.userId,
      dueDate
    });
    
    await task.save();
    await task.populate('assignedTo createdBy', 'name email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.userId },
      { status, updatedAt: Date.now() },
      { new: true }
    ).populate('assignedTo createdBy', 'name email');
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time
router.post('/:id/time-log', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body;
    
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.userId },
      { 
        $push: { 
          timeLogs: { 
            duration, 
            loggedAt: new Date() 
          } 
        },
        $inc: { totalTime: duration }
      },
      { new: true }
    );
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
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
const { authMiddleware } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
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
      createdBy: req.user.userId
    });
    
    await ticket.save();
    await ticket.populate('createdBy', 'name email');
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tickets
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.userId })
      .populate('createdBy assignedTo', 'name email')
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update ticket
router.put('/:id', authMiddleware, async (req, res) => {
  try {
    const { status, priority, assignedTo } = req.body;
    
    const ticket = await Ticket.findByIdAndUpdate(
      req.params.id,
      { status, priority, assignedTo, updatedAt: Date.now() },
      { new: true }
    ).populate('createdBy assignedTo', 'name email');
    
    res.json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
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
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, compose, preprocessing
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

# Models
ticket_classifier = None
anomaly_detector = compose.Pipeline(
    preprocessing.StandardScaler(),
    anomaly.HalfSpaceTrees(seed=42)
)

# Load models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

def load_models():
    global ticket_classifier
    try:
        ticket_classifier = joblib.load(f'{MODEL_PATH}/ticket_classifier.pkl')
    except:
        # Initialize basic classifier if not found
        ticket_classifier = RandomForestClassifier(n_estimators=100)

load_models()

# Request/Response Models
class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    confidence: float

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_logins: int
    session_duration: float
    unusual_access_times: int
    privilege_escalation_attempts: int

class RiskPredictionResponse(BaseModel):
    risk_score: float
    risk_level: str
    factors: List[str]

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_overdue: int
    avg_task_duration: float
    overtime_hours: float
    days_without_break: int

class BurnoutDetectionResponse(BaseModel):
    burnout_risk: str
    score: float
    recommendations: List[str]

# Endpoints
@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    try:
        # Simple keyword-based classification
        text = (request.title + " " + request.description).lower()
        
        if any(word in text for word in ['password', 'login', 'access', 'authentication']):
            return TicketClassificationResponse(category='authentication', confidence=0.85)
        elif any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            return TicketClassificationResponse(category='bug', confidence=0.80)
        elif any(word in text for word in ['feature', 'request', 'enhancement', 'add']):
            return TicketClassificationResponse(category='feature_request', confidence=0.75)
        elif any(word in text for word in ['slow', 'performance', 'speed', 'lag']):
            return TicketClassificationResponse(category='performance', confidence=0.78)
        else:
            return TicketClassificationResponse(category='general', confidence=0.60)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Calculate risk score
        risk_score = 0.0
        factors = []
        
        if request.failed_logins > 3:
            risk_score += 0.3
            factors.append("Multiple failed login attempts")
        
        if request.unusual_access_times > 2:
            risk_score += 0.2
            factors.append("Unusual access times detected")
        
        if request.privilege_escalation_attempts > 0:
            risk_score += 0.4
            factors.append("Privilege escalation attempts")
        
        if request.session_duration > 480:  # 8 hours
            risk_score += 0.1
            factors.append("Unusually long session duration")
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = "high"
        elif risk_score >= 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return RiskPredictionResponse(
            risk_score=min(risk_score, 1.0),
            risk_level=risk_level,
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout", response_model=BurnoutDetectionResponse)
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        # Calculate burnout score
        score = 0.0
        recommendations = []
        
        # Overdue tasks factor
        if request.tasks_overdue > 5:
            score += 0.3
            recommendations.append("Reduce task backlog")
        
        # Overtime factor
        if request.overtime_hours > 10:
            score += 0.25
            recommendations.append("Limit overtime hours")
        
        # Days without break
        if request.days_without_break > 20:
            score += 0.3
            recommendations.append("Take scheduled breaks")
        
        # Task completion rate
        if request.tasks_completed > 0:
            completion_rate = request.tasks_completed / (request.tasks_completed + request.tasks_overdue)
            if completion_rate < 0.6:
                score += 0.15
                recommendations.append("Improve task prioritization")
        
        # Determine risk level
        if score >= 0.7:
            risk = "high"
        elif score >= 0.4:
            risk = "medium"
        else:
            risk = "low"
        
        if not recommendations:
            recommendations.append("Maintain current work-life balance")
        
        return BurnoutDetectionResponse(
            burnout_risk=risk,
            score=min(score, 1.0),
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Frontend Integration

### API Service
```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_URL,
});

const mlApi = axios.create({
  baseURL: ML_API_URL,
});

// Add auth token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authService = {
  login: (email, password) => api.post('/auth/login', { email, password }),
  register: (data) => api.post('/auth/register', data),
  logout: () => localStorage.removeItem('token'),
};

export const userService = {
  getProfile: () => api.get('/users/profile'),
  getAllUsers: () => api.get('/users'),
  updateUser: (id, data) => api.put(`/users/${id}`, data),
  deleteUser: (id) => api.delete(`/users/${id}`),
};

export const taskService = {
  getTasks: () => api.get('/tasks'),
  createTask: (data) => api.post('/tasks', data),
  updateTaskStatus: (id, status) => api.patch(`/tasks/${id}/status`, { status }),
  logTime: (id, duration) => api.post(`/tasks/${id}/time-log`, { duration }),
};

export const ticketService = {
  getTickets: () => api.get('/tickets'),
  createTicket: (data) => api.post('/tickets', data),
  updateTicket: (id, data) => api.put(`/tickets/${id}`, data),
};

export const mlService = {
  classifyTicket: (title, description) => 
    mlApi.post('/classify-ticket', { title, description }),
  predictRisk: (data) => mlApi.post('/predict-risk', data),
  detectBurnout: (data) => mlApi.post('/detect-burnout', data),
};

export default api;
```

### Task Dashboard Component
```javascript
// frontend/src/components/TaskDashboard.jsx
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';

const TaskDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);
  const [filter, setFilter] = useState('all');

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    try {
      const response = await taskService.getTasks();
      setTasks(response.data);
    } catch (error) {
      console.error('Error loading tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await taskService.updateTaskStatus(taskId, newStatus);
      loadTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const getFilteredTasks = (status) => {
    return tasks.filter(task => task.status === status);
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="task-dashboard">
      <h2>My Tasks</h2>
      
      <div className="kanban-board">
        <div className="kanban-column">
          <h3>To Do ({getFilteredTasks('todo').length})</h3>
          {getFilteredTasks('todo').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
        
        <div className="kanban-column">
          <h3>In Progress ({getFilteredTasks('in_progress').length})</h3>
          {getFilteredTasks('in_progress').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
        
        <div className="kanban-column">
          <h3>Done ({getFilteredTasks('done').length})</h3>
          {getFilteredTasks('done').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
      </div>
    </div>
  );
};

const TaskCard = ({ task, onStatusChange }) => {
  return (
    <div className={`task-card priority-${task.priority}`}>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className="priority">{task.priority}</span>
        <span className="due-date">{new Date(task.dueDate).toLocaleDateString()}</span>
      </div>
      <div className="task-actions">
        {task.status === 'todo' && (
          <button onClick={() => onStatusChange(task._id, 'in_progress')}>
            Start
          </button>
        )}
        {task.status === 'in_progress' && (
          <button onClick={() => onStatusChange(task._id, 'done')}>
            Complete
          </button>
        )}
      </div>
    </div>
  );
};

export default TaskDashboard;
```

### AI Analytics Dashboard
```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { mlService, userService } from '../services/api';

const AIAnalytics = () => {
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadAnalytics();
  }, []);

  const loadAnalytics = async () => {
    try {
      const userProfile = await userService.getProfile();
      
      // Get risk prediction
      const riskResponse = await mlService.predictRisk({
        user_id: userProfile.data._id,
        failed_logins: 0,
        session_duration: 120,
        unusual_access_times: 0,
        privilege_escalation_attempts: 0
      });
      setRiskData(riskResponse.data);
      
      // Get burnout detection
      const burnoutResponse = await mlService.detectBurnout({
        user_id: userProfile.data._id,
        tasks_completed: 15,
        tasks_overdue: 3,
        avg_task_duration: 2.5,
        overtime_hours: 5,
        days_without_break: 10
      });
      setBurnoutData(burnoutResponse.data);
    } catch (error) {
      console.error('Error loading analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      <div className="analytics-grid">
        <div className="analytics-card">
          <h3>Risk Assessment</h3>
          <div className={`risk-indicator ${riskData?.risk_level}`}>
            {riskData?.risk_level.toUpperCase()}
          </div>
          <p>Score: {(riskData?.risk_score * 100).toFixed(0)}%</p>
          {riskData?.factors.length > 0 && (
            <div className="risk-factors">
              <h4>Risk Factors:</h4>
              <ul>
                {riskData.factors.map((factor, idx) => (
                  <li key={idx}>{factor}</li>
                ))}
              </ul>
            </div>
          )}
        </div>
        
        <div className="analytics-card">
          <h3>Burnout Detection</h3>
          <div className={`burnout-indicator ${burnoutData?.burnout_risk}`}>
            {burnoutData?.burnout_risk.toUpperCase()}
          </div>
          <p>Score: {(burnoutData?.score * 100).toFixed(0)}%</p>
          {burnoutData?.recommendations.length > 0 && (
            <div className="recommendations">
              <h4>Recommendations:</h4>
              <ul>
                {burnoutData.recommendations.map((rec, idx) => (
                  <li key={idx}>{rec}</li>
                ))}
              </ul>
            </div>
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
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive', 'suspended'], default: 'active' },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', userSchema);
```

### Task Model
```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  status: { type: String, enum: ['todo', 'in_progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high', 'urgent'], default: 'medium' },
  dueDate: { type: Date },
  timeLogs: [{
    duration: Number,
    loggedAt: Date
  }],
  totalTime: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model
```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { type: String },
  priority: { type: String, enum: ['low', 'medium', 'high', 'critical'], default: 'medium' },
  status: { type: String, enum: ['open', 'in_progress', 'resolved', 'closed'], default: 'open' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Configuration

### Backend Server Setup
```javascript
// backend/server.js
const express =
