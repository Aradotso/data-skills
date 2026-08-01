---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, risk detection, and burnout analysis
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "create a task management system with AI insights"
  - "implement ticket classification with machine learning"
  - "build a user dashboard with burnout detection"
  - "set up JWT authentication for enterprise app"
  - "configure AI-powered risk prediction system"
  - "integrate kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack web application for managing users, tasks, and support tickets with integrated AI capabilities including risk detection, anomaly detection, burnout analysis, and predictive project insights. Built with React, Node.js/Express, MongoDB, and FastAPI ML service.

## What This Project Does

This system provides:
- **User Management**: Role-based access control, user CRUD operations, authentication with JWT
- **Task Management**: Kanban board, time tracking, task assignment and monitoring
- **Support Tickets**: AI-powered classification and routing, ticket lifecycle management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Admin analytics, user performance tracking, audit logs

## Installation

### Prerequisites
- Node.js 14+
- Python 3.8+
- MongoDB

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
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
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
```env
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
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
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Key Architecture

### Backend API Structure

**Authentication Routes** (`/api/auth`)
```javascript
// backend/routes/authRoutes.js
const express = require('express');
const router = express.Router();
const authController = require('../controllers/authController');

router.post('/register', authController.register);
router.post('/login', authController.login);
router.get('/me', authController.protect, authController.getMe);

module.exports = router;
```

**User Routes** (`/api/users`)
```javascript
// backend/routes/userRoutes.js
const express = require('express');
const router = express.Router();
const userController = require('../controllers/userController');
const { protect, authorize } = require('../middleware/auth');

router.use(protect); // All routes require authentication

router.get('/', authorize('admin'), userController.getUsers);
router.post('/', authorize('admin'), userController.createUser);
router.get('/:id', userController.getUser);
router.put('/:id', authorize('admin'), userController.updateUser);
router.delete('/:id', authorize('admin'), userController.deleteUser);

module.exports = router;
```

**Task Routes** (`/api/tasks`)
```javascript
// backend/routes/taskRoutes.js
const express = require('express');
const router = express.Router();
const taskController = require('../controllers/taskController');
const { protect } = require('../middleware/auth');

router.use(protect);

router.get('/', taskController.getTasks);
router.post('/', taskController.createTask);
router.get('/:id', taskController.getTask);
router.put('/:id', taskController.updateTask);
router.delete('/:id', taskController.deleteTask);
router.put('/:id/status', taskController.updateTaskStatus);
router.post('/:id/time-log', taskController.logTime);

module.exports = router;
```

**Ticket Routes** (`/api/tickets`)
```javascript
// backend/routes/ticketRoutes.js
const express = require('express');
const router = express.Router();
const ticketController = require('../controllers/ticketController');
const { protect, authorize } = require('../middleware/auth');

router.use(protect);

router.get('/', ticketController.getTickets);
router.post('/', ticketController.createTicket);
router.get('/:id', ticketController.getTicket);
router.put('/:id', ticketController.updateTicket);
router.put('/:id/assign', authorize('admin'), ticketController.assignTicket);
router.post('/:id/classify', ticketController.classifyTicket); // AI classification

module.exports = router;
```

## Core Implementations

### Authentication with JWT

```javascript
// backend/controllers/authController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Generate JWT token
const signToken = (id) => {
  return jwt.sign({ id }, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_EXPIRE
  });
};

exports.register = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'Email already registered' });
    }
    
    // Create user
    const user = await User.create({
      name,
      email,
      password,
      role: role || 'user'
    });
    
    // Generate token
    const token = signToken(user._id);
    
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
    res.status(500).json({ error: error.message });
  }
};

exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;
    
    // Validate input
    if (!email || !password) {
      return res.status(400).json({ error: 'Please provide email and password' });
    }
    
    // Check user and password
    const user = await User.findOne({ email }).select('+password');
    if (!user || !(await user.comparePassword(password))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // Generate token
    const token = signToken(user._id);
    
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
    res.status(500).json({ error: error.message });
  }
};
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  try {
    let token;
    
    if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
      token = req.headers.authorization.split(' ')[1];
    }
    
    if (!token) {
      return res.status(401).json({ error: 'Not authorized to access this route' });
    }
    
    // Verify token
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    // Get user from token
    req.user = await User.findById(decoded.id);
    
    if (!req.user) {
      return res.status(401).json({ error: 'User not found' });
    }
    
    next();
  } catch (error) {
    res.status(401).json({ error: 'Not authorized to access this route' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        error: `User role ${req.user.role} is not authorized to access this route` 
      });
    }
    next();
  };
};
```

### User Model with Password Hashing

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
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
    match: [/^\S+@\S+\.\S+$/, 'Please provide a valid email']
  },
  password: {
    type: String,
    required: [true, 'Please provide a password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  position: String,
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
  
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const timeLogSchema = new mongoose.Schema({
  startTime: Date,
  endTime: Date,
  duration: Number, // in minutes
  loggedAt: {
    type: Date,
    default: Date.now
  }
});

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: String,
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
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
  dueDate: Date,
  timeLogs: [timeLogSchema],
  totalTimeSpent: {
    type: Number,
    default: 0
  },
  tags: [String],
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
});

taskSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
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
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'feature-request', 'bug'],
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
  aiClassification: {
    category: String,
    confidence: Number,
    suggestedAssignee: String
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## AI/ML Service Integration

### FastAPI ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib
import os

app = FastAPI(title="Enterprise UMS ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    confidence: float
    suggestedPriority: str

class RiskPredictionRequest(BaseModel):
    userId: str
    failedLogins: int
    unusualActivityCount: int
    lastLoginHoursAgo: float

class RiskPredictionResponse(BaseModel):
    riskScore: float
    riskLevel: str
    reasons: List[str]

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    tasksCompleted: int
    averageWorkHours: float
    overtimeHours: float
    tasksOverdue: int

class BurnoutAnalysisResponse(BaseModel):
    burnoutScore: float
    burnoutLevel: str
    recommendations: List[str]

# Ticket Classification
@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    try:
        # Combine title and description
        text = f"{request.title} {request.description}".lower()
        
        # Simple keyword-based classification
        categories = {
            'technical': ['error', 'bug', 'crash', 'not working', 'issue', 'problem'],
            'billing': ['payment', 'invoice', 'charge', 'refund', 'subscription'],
            'feature-request': ['feature', 'request', 'add', 'new', 'enhancement'],
            'bug': ['bug', 'broken', 'error', 'crash', 'fail']
        }
        
        scores = {}
        for category, keywords in categories.items():
            score = sum(1 for keyword in keywords if keyword in text)
            scores[category] = score
        
        # Get category with highest score
        predicted_category = max(scores, key=scores.get) if max(scores.values()) > 0 else 'general'
        confidence = min(max(scores.values()) / len(categories.get(predicted_category, [])), 1.0) if max(scores.values()) > 0 else 0.5
        
        # Suggest priority based on keywords
        urgent_keywords = ['urgent', 'critical', 'asap', 'immediately', 'down', 'crash']
        high_keywords = ['important', 'soon', 'error', 'bug']
        
        if any(keyword in text for keyword in urgent_keywords):
            priority = 'critical'
        elif any(keyword in text for keyword in high_keywords):
            priority = 'high'
        else:
            priority = 'medium'
        
        return TicketClassificationResponse(
            category=predicted_category,
            confidence=confidence,
            suggestedPriority=priority
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Risk Prediction
@app.post("/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Calculate risk score based on factors
        risk_score = 0.0
        reasons = []
        
        # Failed logins
        if request.failedLogins > 5:
            risk_score += 0.3
            reasons.append(f"High number of failed login attempts ({request.failedLogins})")
        elif request.failedLogins > 2:
            risk_score += 0.15
            reasons.append(f"Multiple failed login attempts ({request.failedLogins})")
        
        # Unusual activity
        if request.unusualActivityCount > 10:
            risk_score += 0.4
            reasons.append(f"Unusual activity detected ({request.unusualActivityCount} events)")
        elif request.unusualActivityCount > 5:
            risk_score += 0.2
            reasons.append(f"Some unusual activity detected ({request.unusualActivityCount} events)")
        
        # Last login time
        if request.lastLoginHoursAgo > 720:  # 30 days
            risk_score += 0.2
            reasons.append("Account inactive for extended period")
        
        # Normalize risk score
        risk_score = min(risk_score, 1.0)
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = "critical"
        elif risk_score >= 0.5:
            risk_level = "high"
        elif risk_score >= 0.3:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        if not reasons:
            reasons.append("No significant risk factors detected")
        
        return RiskPredictionResponse(
            riskScore=round(risk_score, 2),
            riskLevel=risk_level,
            reasons=reasons
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout Analysis
@app.post("/analyze-burnout", response_model=BurnoutAnalysisResponse)
async def analyze_burnout(request: BurnoutAnalysisRequest):
    try:
        burnout_score = 0.0
        recommendations = []
        
        # Work hours analysis
        if request.averageWorkHours > 50:
            burnout_score += 0.3
            recommendations.append("Reduce working hours to maintain work-life balance")
        elif request.averageWorkHours > 45:
            burnout_score += 0.15
        
        # Overtime analysis
        if request.overtimeHours > 20:
            burnout_score += 0.25
            recommendations.append("Excessive overtime detected - consider task redistribution")
        elif request.overtimeHours > 10:
            burnout_score += 0.1
        
        # Overdue tasks
        if request.tasksOverdue > 5:
            burnout_score += 0.3
            recommendations.append("High number of overdue tasks - may need support or deadline adjustment")
        elif request.tasksOverdue > 2:
            burnout_score += 0.15
        
        # Task completion rate
        if request.tasksCompleted < 5 and request.tasksOverdue > 3:
            burnout_score += 0.15
            recommendations.append("Low task completion with high overdue count - possible overwhelm")
        
        # Normalize
        burnout_score = min(burnout_score, 1.0)
        
        # Determine level
        if burnout_score >= 0.7:
            burnout_level = "critical"
            recommendations.append("Immediate intervention recommended")
        elif burnout_score >= 0.5:
            burnout_level = "high"
            recommendations.append("Close monitoring and support needed")
        elif burnout_score >= 0.3:
            burnout_level = "moderate"
        else:
            burnout_level = "low"
        
        if not recommendations:
            recommendations.append("Healthy work patterns maintained")
        
        return BurnoutAnalysisResponse(
            burnoutScore=round(burnout_score, 2),
            burnoutLevel=burnout_level,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "Enterprise UMS ML Service"}
```

### Calling ML Service from Backend

```javascript
// backend/services/mlService.js
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL || 'http://localhost:8000';

exports.classifyTicket = async (title, description) => {
  try {
    const response = await axios.post(`${ML_SERVICE_URL}/classify-ticket`, {
      title,
      description
    });
    return response.data;
  } catch (error) {
    console.error('ML Service Error:', error.message);
    return null;
  }
};

exports.predictRisk = async (userId, failedLogins, unusualActivityCount, lastLoginHoursAgo) => {
  try {
    const response = await axios.post(`${ML_SERVICE_URL}/predict-risk`, {
      userId,
      failedLogins,
      unusualActivityCount,
      lastLoginHoursAgo
    });
    return response.data;
  } catch (error) {
    console.error('ML Service Error:', error.message);
    return null;
  }
};

exports.analyzeBurnout = async (userId, tasksCompleted, averageWorkHours, overtimeHours, tasksOverdue) => {
  try {
    const response = await axios.post(`${ML_SERVICE_URL}/analyze-burnout`, {
      userId,
      tasksCompleted,
      averageWorkHours,
      overtimeHours,
      tasksOverdue
    });
    return response.data;
  } catch (error) {
    console.error('ML Service Error:', error.message);
    return null;
  }
};
```

### Ticket Controller with AI Classification

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const mlService = require('../services/mlService');

exports.createTicket = async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Get AI classification
    const aiClassification = await mlService.classifyTicket(title, description);
    
    const ticket = await Ticket.create({
      title,
      description,
      priority: priority || aiClassification?.suggestedPriority || 'medium',
      category: aiClassification?.category || 'general',
      createdBy: req.user._id,
      aiClassification: aiClassification ? {
        category: aiClassification.category,
        confidence: aiClassification.confidence,
        suggestedAssignee: null // Can be extended
      } : null
    });
    
    await ticket.populate('createdBy', 'name email');
    
    res.status(201).json({
      success: true,
      ticket
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.getTickets = async (req, res) => {
  try {
    const { status, category, priority } = req.query;
    
    const filter = {};
    if (status) filter.status = status;
    if (category) filter.category = category;
    if (priority) filter.priority = priority;
    
    // Non-admin users only see their tickets
    if (req.user.role !== 'admin') {
      filter.$or = [
        { createdBy: req.user._id },
        { assignedTo: req.user._id }
      ];
    }
    
    const tickets = await Ticket.find(filter)
      .populate('createdBy', 'name email')
      .populate('assignedTo', 'name email')
      .sort({ createdAt: -1 });
    
    res.json({
      success: true,
      count: tickets.length,
      tickets
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.classifyTicket = async (req, res) => {
  try {
    const ticket = await Ticket.findById(req.params.id);
    
    if (!ticket) {
      return res.status(404).json({ error: 'Ticket not found' });
    }
    
    const classification = await mlService.classifyTicket(ticket.title, ticket.description);
    
    if (classification) {
      ticket.category = classification.category;
      ticket.aiClassification = {
        category: classification.category,
        confidence: classification.confidence,
        suggestedAssignee: null
      };
      await ticket.save();
    }
    
    res.json({
      success: true,
      classification,
      ticket
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

## Frontend Integration

### API Service Setup

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';

// Create axios instance
const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to requests
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// Auth API
export const authAPI = {
  login: (email, password) => api.post('/auth/login', { email, password }),
  register: (userData) => api.post('/auth/register', userData),
  getMe: () => api.get('/auth/me')
};

// Users API
export const usersAPI = {
  getAll: () => api.get('/users'),
  getById: (id) => api.get(`/users/${id}`),
  create: (userData) => api.post('/users', userData),
  update: (id, userData) => api.put(`/users/${id}`, userData),
  delete: (id) => api.delete(`/users/${id}`)
};

// Tasks API
export const tasksAPI = {
  getAll: (params) => api.get('/tasks', { params }),
  getById: (id) => api.get(`/tasks/${id}`),
  create: (taskData) => api.post('/tasks', taskData),
  update: (id, taskData) => api.put(`/tasks/${id}`, taskData),
  updateStatus: (id, status) => api.put(`/tasks/${id}/status`, { status }),
  logTime: (id, startTime, endTime) => api.post(`/tasks/${id}/time-log`, { startTime, endTime }),
  delete: (id) => api.delete(`/tasks/${id}`)
};

// Tickets API
export const ticketsAPI = {
  getAll: (params) => api.get('/tickets', { params }),
  getById: (id) => api.get(`/tickets/${id}`),
  create: (ticketData) => api.post('/tickets', ticketData),
  update: (id, ticketData) => api.put(`/tickets/${id}`, ticketData),
  classify: (id) => api.post(`/tickets/${id}/classify`),
  assign: (id, assigneeId) => api.put(`/tickets/${id}/assign`, { assignedTo: assigneeId })
};

export default api;
```

### React Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';
import { authAPI } from '../services/api';

const AuthContext = createContext();

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      fetchUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const response = await authAPI.getMe();
      setUser(response.data.user);
    } catch (error) {
      console.error('Failed to fetch user:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    try {
      const response = await authAPI.login(email, password);
      const { token, user } = response.data;
      localStorage.setItem('token', token);
      setToken(token);
      setUser(user);
      return { success: true };
    } catch (error) {
      return { 
        success: false, 
        error: error.response?.data?.error || 'Login failed' 
      };
    }
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  const register = async (userData) => {
    try {
      const response = await authAPI.register(userData);
      const { token, user } = response.data;
      localStorage.setItem('token', token);
      setToken(token);
      setUser(user);
      return { success: true };
    } catch (error) {
      return { 
        success: false
