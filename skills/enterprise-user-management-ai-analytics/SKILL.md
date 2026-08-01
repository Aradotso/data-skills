---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, task management, and automated ticket routing
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement AI-powered ticket classification system"
  - "create user management dashboard with risk detection"
  - "build task management with burnout analysis"
  - "integrate AI analytics for enterprise users"
  - "deploy user management system with ML insights"
  - "configure AI-driven support ticket routing"
  - "add predictive analytics to user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application combining user administration, task tracking, and support ticket management with AI-powered analytics. The system provides risk prediction, anomaly detection, burnout analysis, and automated ticket classification using machine learning models built with scikit-learn and River.

The architecture consists of three main components:
- **Frontend**: React.js dashboard for admins and users
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based microservice for AI analytics

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB (local or cloud instance)

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

Create `.env` file in backend directory:
```env
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

Create `.env` file in ml-service directory:
```env
MONGODB_URI=your_mongodb_connection_string
MODEL_PATH=./models
```

Start ML service:
```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Backend API Usage

### Authentication

**User Login**:
```javascript
// JavaScript (Node.js backend)
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('./models/User');

const router = express.Router();

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
      { expiresIn: '24h' }
    );
    
    res.json({ token, user: { id: user._id, email: user.email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

**Authentication Middleware**:
```javascript
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ message: 'No token provided' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
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

### User Management

**Create User (Admin)**:
```javascript
const { authMiddleware, adminOnly } = require('./middleware/auth');

router.post('/users', authMiddleware, adminOnly, async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const hashedPassword = await bcrypt.hash(password, 10);
    
    const user = new User({
      name,
      email,
      password: hashedPassword,
      role: role || 'user',
      department,
      createdBy: req.user.userId
    });
    
    await user.save();
    
    // Log audit trail
    await AuditLog.create({
      action: 'USER_CREATED',
      performedBy: req.user.userId,
      targetUser: user._id,
      timestamp: new Date()
    });
    
    res.status(201).json({ message: 'User created', userId: user._id });
  } catch (error) {
    res.status(500).json({ message: 'Error creating user', error: error.message });
  }
});
```

**Get All Users**:
```javascript
router.get('/users', authMiddleware, adminOnly, async (req, res) => {
  try {
    const { page = 1, limit = 10, department, role } = req.query;
    
    const filter = {};
    if (department) filter.department = department;
    if (role) filter.role = role;
    
    const users = await User.find(filter)
      .select('-password')
      .limit(limit * 1)
      .skip((page - 1) * limit)
      .sort({ createdAt: -1 });
    
    const count = await User.countDocuments(filter);
    
    res.json({
      users,
      totalPages: Math.ceil(count / limit),
      currentPage: page
    });
  } catch (error) {
    res.status(500).json({ message: 'Error fetching users', error: error.message });
  }
});
```

### Task Management

**Create Task**:
```javascript
const Task = require('./models/Task');

router.post('/tasks', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.userId,
      priority: priority || 'medium',
      status: 'todo',
      dueDate: new Date(dueDate),
      createdAt: new Date()
    });
    
    await task.save();
    
    res.status(201).json({ message: 'Task created', taskId: task._id });
  } catch (error) {
    res.status(500).json({ message: 'Error creating task', error: error.message });
  }
});
```

**Update Task Status**:
```javascript
router.patch('/tasks/:taskId/status', authMiddleware, async (req, res) => {
  try {
    const { taskId } = req.params;
    const { status } = req.body; // 'todo', 'in_progress', 'done'
    
    const task = await Task.findById(taskId);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    // Check if user is assigned to task or is admin
    if (task.assignedTo.toString() !== req.user.userId && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Not authorized' });
    }
    
    task.status = status;
    if (status === 'in_progress' && !task.startedAt) {
      task.startedAt = new Date();
    }
    if (status === 'done' && !task.completedAt) {
      task.completedAt = new Date();
    }
    
    await task.save();
    
    res.json({ message: 'Task updated', task });
  } catch (error) {
    res.status(500).json({ message: 'Error updating task', error: error.message });
  }
});
```

### Support Ticket Management

**Create Ticket**:
```javascript
const Ticket = require('./models/Ticket');

router.post('/tickets', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    const ticket = new Ticket({
      title,
      description,
      priority: priority || 'medium',
      status: 'open',
      createdBy: req.user.userId,
      createdAt: new Date()
    });
    
    await ticket.save();
    
    // Call ML service for classification
    const mlResponse = await fetch(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        title,
        description,
        ticketId: ticket._id
      })
    });
    
    const mlData = await mlResponse.json();
    
    ticket.category = mlData.category;
    ticket.suggestedAssignee = mlData.suggestedAssignee;
    await ticket.save();
    
    res.status(201).json({ message: 'Ticket created', ticket });
  } catch (error) {
    res.status(500).json({ message: 'Error creating ticket', error: error.message });
  }
});
```

## ML Service API

### Ticket Classification

**FastAPI Endpoint**:
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import pickle
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import os

app = FastAPI()

class TicketRequest(BaseModel):
    title: str
    description: str
    ticketId: str

class TicketResponse(BaseModel):
    category: str
    suggestedAssignee: Optional[str]
    confidence: float

# Load pre-trained model
MODEL_PATH = os.getenv('MODEL_PATH', './models')

try:
    with open(f'{MODEL_PATH}/ticket_classifier.pkl', 'rb') as f:
        classifier = pickle.load(f)
    with open(f'{MODEL_PATH}/vectorizer.pkl', 'rb') as f:
        vectorizer = pickle.load(f)
except FileNotFoundError:
    classifier = None
    vectorizer = None

@app.post('/classify-ticket', response_model=TicketResponse)
async def classify_ticket(request: TicketRequest):
    if not classifier or not vectorizer:
        raise HTTPException(status_code=503, detail='Model not loaded')
    
    # Combine title and description
    text = f"{request.title} {request.description}"
    
    # Vectorize and predict
    features = vectorizer.transform([text])
    category = classifier.predict(features)[0]
    confidence = max(classifier.predict_proba(features)[0])
    
    # Simple rule-based assignee suggestion
    assignee_map = {
        'technical': 'tech_support_team',
        'billing': 'finance_team',
        'account': 'customer_service',
        'general': 'support_team'
    }
    
    return TicketResponse(
        category=category,
        suggestedAssignee=assignee_map.get(category, 'support_team'),
        confidence=float(confidence)
    )
```

### Risk Prediction

**Risk Detection Model**:
```python
from pydantic import BaseModel
from typing import List
import numpy as np

class UserBehavior(BaseModel):
    userId: str
    loginFrequency: int
    taskCompletionRate: float
    avgTaskDuration: float
    failedLoginAttempts: int
    unusualAccessTimes: int

class RiskResponse(BaseModel):
    riskScore: float
    riskLevel: str
    factors: List[str]

@app.post('/predict-risk', response_model=RiskResponse)
async def predict_risk(behavior: UserBehavior):
    # Calculate risk score based on multiple factors
    risk_factors = []
    risk_score = 0.0
    
    # Failed login attempts
    if behavior.failedLoginAttempts > 3:
        risk_score += 0.3
        risk_factors.append('Multiple failed login attempts')
    
    # Task completion rate
    if behavior.taskCompletionRate < 0.5:
        risk_score += 0.2
        risk_factors.append('Low task completion rate')
    
    # Unusual access times
    if behavior.unusualAccessTimes > 5:
        risk_score += 0.25
        risk_factors.append('Unusual access patterns')
    
    # Login frequency anomaly
    if behavior.loginFrequency < 1 or behavior.loginFrequency > 50:
        risk_score += 0.25
        risk_factors.append('Abnormal login frequency')
    
    # Determine risk level
    if risk_score >= 0.7:
        risk_level = 'high'
    elif risk_score >= 0.4:
        risk_level = 'medium'
    else:
        risk_level = 'low'
    
    return RiskResponse(
        riskScore=min(risk_score, 1.0),
        riskLevel=risk_level,
        factors=risk_factors
    )
```

### Burnout Detection

**Burnout Analysis**:
```python
from datetime import datetime, timedelta

class WorkloadData(BaseModel):
    userId: str
    tasksAssigned: int
    tasksCompleted: int
    avgWorkHoursPerDay: float
    daysWithoutBreak: int
    overtimeHours: float

class BurnoutResponse(BaseModel):
    burnoutScore: float
    burnoutRisk: str
    recommendations: List[str]

@app.post('/detect-burnout', response_model=BurnoutResponse)
async def detect_burnout(workload: WorkloadData):
    score = 0.0
    recommendations = []
    
    # High task load
    if workload.tasksAssigned > 15:
        score += 0.25
        recommendations.append('Consider redistributing tasks')
    
    # Work hours
    if workload.avgWorkHoursPerDay > 10:
        score += 0.3
        recommendations.append('Reduce daily work hours')
    
    # No breaks
    if workload.daysWithoutBreak > 10:
        score += 0.25
        recommendations.append('Schedule time off')
    
    # Overtime
    if workload.overtimeHours > 20:
        score += 0.2
        recommendations.append('Limit overtime hours')
    
    # Task completion stress
    completion_rate = workload.tasksCompleted / max(workload.tasksAssigned, 1)
    if completion_rate < 0.6:
        score += 0.15
        recommendations.append('Provide additional support or training')
    
    if score >= 0.7:
        risk = 'high'
    elif score >= 0.4:
        risk = 'moderate'
    else:
        risk = 'low'
    
    return BurnoutResponse(
        burnoutScore=min(score, 1.0),
        burnoutRisk=risk,
        recommendations=recommendations
    )
```

## Frontend Integration

### React API Client

**API Service**:
```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

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

export const authService = {
  login: async (email, password) => {
    const response = await api.post('/login', { email, password });
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
    }
    return response.data;
  },
  
  logout: () => {
    localStorage.removeItem('token');
  }
};

export const userService = {
  getUsers: async (page = 1, filters = {}) => {
    const response = await api.get('/users', { params: { page, ...filters } });
    return response.data;
  },
  
  createUser: async (userData) => {
    const response = await api.post('/users', userData);
    return response.data;
  },
  
  updateUser: async (userId, userData) => {
    const response = await api.patch(`/users/${userId}`, userData);
    return response.data;
  },
  
  deleteUser: async (userId) => {
    const response = await api.delete(`/users/${userId}`);
    return response.data;
  }
};

export const taskService = {
  getTasks: async (userId) => {
    const response = await api.get('/tasks', { params: { userId } });
    return response.data;
  },
  
  createTask: async (taskData) => {
    const response = await api.post('/tasks', taskData);
    return response.data;
  },
  
  updateTaskStatus: async (taskId, status) => {
    const response = await api.patch(`/tasks/${taskId}/status`, { status });
    return response.data;
  }
};

export const ticketService = {
  createTicket: async (ticketData) => {
    const response = await api.post('/tickets', ticketData);
    return response.data;
  },
  
  getTickets: async (filters = {}) => {
    const response = await api.get('/tickets', { params: filters });
    return response.data;
  }
};

export const mlService = {
  getRiskScore: async (userId, behaviorData) => {
    const response = await axios.post(`${ML_API_URL}/predict-risk`, behaviorData);
    return response.data;
  },
  
  getBurnoutAnalysis: async (userId, workloadData) => {
    const response = await axios.post(`${ML_API_URL}/detect-burnout`, workloadData);
    return response.data;
  }
};
```

### Task Board Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    in_progress: [],
    done: []
  });

  useEffect(() => {
    loadTasks();
  }, [userId]);

  const loadTasks = async () => {
    try {
      const data = await taskService.getTasks(userId);
      const grouped = {
        todo: data.filter(t => t.status === 'todo'),
        in_progress: data.filter(t => t.status === 'in_progress'),
        done: data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error loading tasks:', error);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskService.updateTaskStatus(taskId, newStatus);
      loadTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskColumn = ({ title, status, tasks }) => (
    <div className="task-column">
      <h3>{title}</h3>
      {tasks.map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority-${task.priority}`}>{task.priority}</span>
          <select 
            value={status} 
            onChange={(e) => handleStatusChange(task._id, e.target.value)}
          >
            <option value="todo">To Do</option>
            <option value="in_progress">In Progress</option>
            <option value="done">Done</option>
          </select>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      <TaskColumn title="To Do" status="todo" tasks={tasks.todo} />
      <TaskColumn title="In Progress" status="in_progress" tasks={tasks.in_progress} />
      <TaskColumn title="Done" status="done" tasks={tasks.done} />
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import { mlService } from '../services/api';

const AIAnalyticsDashboard = ({ userId, userBehavior, workloadData }) => {
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);

  useEffect(() => {
    loadAnalytics();
  }, [userId]);

  const loadAnalytics = async () => {
    try {
      const [risk, burnout] = await Promise.all([
        mlService.getRiskScore(userId, userBehavior),
        mlService.getBurnoutAnalysis(userId, workloadData)
      ]);
      
      setRiskData(risk);
      setBurnoutData(burnout);
    } catch (error) {
      console.error('Error loading analytics:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <div className="risk-panel">
        <h3>Security Risk Assessment</h3>
        {riskData && (
          <>
            <div className={`risk-score ${riskData.riskLevel}`}>
              <span className="score">{(riskData.riskScore * 100).toFixed(0)}%</span>
              <span className="level">{riskData.riskLevel.toUpperCase()}</span>
            </div>
            <ul className="risk-factors">
              {riskData.factors.map((factor, idx) => (
                <li key={idx}>{factor}</li>
              ))}
            </ul>
          </>
        )}
      </div>

      <div className="burnout-panel">
        <h3>Burnout Analysis</h3>
        {burnoutData && (
          <>
            <div className={`burnout-score ${burnoutData.burnoutRisk}`}>
              <span className="score">{(burnoutData.burnoutScore * 100).toFixed(0)}%</span>
              <span className="risk">{burnoutData.burnoutRisk.toUpperCase()} RISK</span>
            </div>
            <div className="recommendations">
              <h4>Recommendations:</h4>
              <ul>
                {burnoutData.recommendations.map((rec, idx) => (
                  <li key={idx}>{rec}</li>
                ))}
              </ul>
            </div>
          </>
        )}
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Database Models

### User Schema

```javascript
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user', 'manager'], default: 'user' },
  department: { type: String },
  isActive: { type: Boolean, default: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  lastLogin: { type: Date },
  failedLoginAttempts: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', userSchema);
```

### Task Schema

```javascript
const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  status: { type: String, enum: ['todo', 'in_progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high', 'urgent'], default: 'medium' },
  dueDate: { type: Date },
  startedAt: { type: Date },
  completedAt: { type: Date },
  timeTracked: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Schema

```javascript
const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  status: { type: String, enum: ['open', 'in_progress', 'resolved', 'closed'], default: 'open' },
  priority: { type: String, enum: ['low', 'medium', 'high', 'urgent'], default: 'medium' },
  category: { type: String }, // AI-generated
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  suggestedAssignee: { type: String }, // AI suggestion
  resolution: { type: String },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: { type: Date },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Training ML Models

### Ticket Classification Model

```python
# ml-service/train_classifier.py
import pandas as pd
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.model_selection import train_test_split
import pickle
import os

def train_ticket_classifier():
    # Load training data from MongoDB or CSV
    # Example structure: tickets with 'text' and 'category' columns
    
    # Sample data (in production, load from database)
    data = {
        'text': [
            'Cannot login to my account',
            'Need help with billing invoice',
            'System is running slow',
            'Forgot my password',
            'Question about subscription plan',
            'Server error 500',
            'Payment failed',
            'Feature request for new dashboard'
        ],
        'category': [
            'account', 'billing', 'technical', 'account',
            'billing', 'technical', 'billing', 'general'
        ]
    }
    
    df = pd.DataFrame(data)
    
    # Vectorize text
    vectorizer = TfidfVectorizer(max_features=1000, stop_words='english')
    X = vectorizer.fit_transform(df['text'])
    y = df['category']
    
    # Train classifier
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
    
    classifier = MultinomialNB()
    classifier.fit(X_train, y_train)
    
    # Evaluate
    accuracy = classifier.score(X_test, y_test)
    print(f'Classifier accuracy: {accuracy:.2f}')
    
    # Save models
    model_path = os.getenv('MODEL_PATH', './models')
    os.makedirs(model_path, exist_ok=True)
    
    with open(f'{model_path}/ticket_classifier.pkl', 'wb') as f:
        pickle.dump(classifier, f)
    
    with open(f'{model_path}/vectorizer.pkl', 'wb') as f:
        pickle.dump(vectorizer, f)
    
    print('Models saved successfully')

if __name__ == '__main__':
    train_ticket_classifier()
```

Run training:
```bash
cd ml-service
python train_classifier.py
```

## Common Patterns

### Role-Based Access Control

```javascript
// Middleware for different role levels
const requireRole = (roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: 'Access denied' });
    }
    next();
  };
};

// Usage
router.get('/admin/analytics', 
  authMiddleware, 
  requireRole(['admin', 'manager']), 
  async (req, res) => {
    // Admin analytics
  }
);
```

### Audit Logging

```javascript
const AuditLog = require('./models/AuditLog');

const logAction = async (action, userId, targetId, details = {}) => {
  await AuditLog.create({
    action,
    performedBy: userId,
    targetId,
    details,
    timestamp: new Date(),
    ipAddress: details.ip || null
  });
};

// Usage in routes
router.delete('/users/:userId', authMiddleware, adminOnly, async (req, res) => {
  const { userId } = req.params;
  
  await User.findByIdAndDelete(userId);
  
  await logAction('USER_DELETED', req.user.userId, userId, {
    ip: req.ip
  });
  
  res.json({ message: 'User deleted' });
});
```

### Real-time Notifications

```javascript
// Using Socket.io for real-time updates
const io = require('socket.io')(server);

io.on('connection', (socket) => {
  socket.on('join', (userId) => {
    socket.join
