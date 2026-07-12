---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and predictive insights
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics into user management
  - implement user task tracking with AI insights
  - create admin dashboard with user analytics
  - build ticket classification system with ML
  - add burnout detection to user management
  - configure AI-powered risk prediction
  - set up kanban board with time tracking
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-powered analytics. It provides:

- **User Management**: JWT authentication, role-based access control (RBAC), user CRUD operations
- **Task Management**: Kanban boards (To Do/In Progress/Done), time tracking, assignments
- **Support System**: Ticket creation, AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organizational analytics, audit logs, security alerts

The system uses a microservices architecture with a React frontend, Node.js/Express backend, and FastAPI-based ML service.

## Installation

### Prerequisites

```bash
# Ensure you have these installed
node --version  # v14+
npm --version   # v6+
python --version # 3.8+
mongod --version # MongoDB 4.4+
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
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=${MONGODB_URI}
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
EOF

npm start
```

## Key API Endpoints

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register user
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const user = await User.create({
      name,
      email,
      password, // Hashed in pre-save hook
      role: role || 'user'
    });
    
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
    res.status(500).json({ message: error.message });
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
    
    const isMatch = await user.comparePassword(password);
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
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### User Management (Admin)

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { protect, authorize } = require('../middleware/auth');

// Get all users (admin only)
router.get('/', protect, authorize('admin'), async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({
      success: true,
      count: users.length,
      data: users
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update user
router.put('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Delete user
router.delete('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json({ success: true, message: 'User deleted' });
  } catch (error) {
    res.status(500).json({ message: error.message });
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
const { protect } = require('../middleware/auth');

// Get user tasks
router.get('/my-tasks', protect, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'name email')
      .sort('-createdAt');
    
    res.json({ success: true, data: tasks });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = await Task.create({
      title,
      description,
      assignedTo,
      assignedBy: req.user.id,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body; // 'todo', 'inprogress', 'done'
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    // Check if user owns the task
    if (task.assignedTo.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Not authorized' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = Date.now();
    }
    await task.save();
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Track time
router.post('/:id/time', protect, async (req, res) => {
  try {
    const { duration } = req.body; // in seconds
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.timeSpent = (task.timeSpent || 0) + duration;
    await task.save();
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Support Tickets

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const axios = require('axios');
const { protect } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    let category = 'general';
    let aiPriority = priority;
    
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { title, description }
      );
      category = mlResponse.data.category;
      aiPriority = mlResponse.data.priority;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    const ticket = await Ticket.create({
      title,
      description,
      priority: aiPriority,
      category,
      createdBy: req.user.id,
      status: 'open'
    });
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', protect, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .sort('-createdAt');
    
    res.json({ success: true, data: tickets });
  } catch (error) {
    res.status(500).json({ message: error.message });
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
from typing import Optional
import joblib
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import os

app = FastAPI()

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketInput(BaseModel):
    title: str
    description: str

class TicketClassification(BaseModel):
    category: str
    priority: str
    confidence: float

# Simple keyword-based classifier (replace with trained model)
CATEGORY_KEYWORDS = {
    'technical': ['error', 'bug', 'crash', 'not working', 'broken', 'issue'],
    'access': ['login', 'password', 'access', 'permission', 'locked'],
    'feature': ['request', 'add', 'new', 'feature', 'enhancement'],
    'general': []
}

PRIORITY_KEYWORDS = {
    'high': ['urgent', 'critical', 'emergency', 'asap', 'down'],
    'medium': ['important', 'soon', 'needed'],
    'low': ['minor', 'whenever', 'suggestion']
}

@app.post("/classify-ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketInput):
    text = f"{ticket.title} {ticket.description}".lower()
    
    # Classify category
    category = 'general'
    max_matches = 0
    for cat, keywords in CATEGORY_KEYWORDS.items():
        matches = sum(1 for kw in keywords if kw in text)
        if matches > max_matches:
            max_matches = matches
            category = cat
    
    # Classify priority
    priority = 'low'
    for pri, keywords in PRIORITY_KEYWORDS.items():
        if any(kw in text for kw in keywords):
            priority = pri
            break
    
    confidence = min(0.95, 0.6 + (max_matches * 0.1))
    
    return TicketClassification(
        category=category,
        priority=priority,
        confidence=confidence
    )
```

### Risk Prediction

```python
# ml-service/main.py (continued)
from datetime import datetime, timedelta
import motor.motor_asyncio
from bson import ObjectId

class UserBehavior(BaseModel):
    user_id: str
    failed_logins: int = 0
    unusual_access_times: int = 0
    data_access_volume: int = 0
    permission_changes: int = 0

class RiskScore(BaseModel):
    risk_level: str  # low, medium, high
    score: float  # 0-100
    factors: list[str]
    recommendations: list[str]

@app.post("/predict-risk", response_model=RiskScore)
async def predict_risk(behavior: UserBehavior):
    score = 0
    factors = []
    
    # Failed login attempts
    if behavior.failed_logins > 5:
        score += 30
        factors.append(f"High failed login attempts: {behavior.failed_logins}")
    elif behavior.failed_logins > 2:
        score += 15
        factors.append(f"Moderate failed login attempts: {behavior.failed_logins}")
    
    # Unusual access times
    if behavior.unusual_access_times > 10:
        score += 25
        factors.append(f"Unusual access times detected: {behavior.unusual_access_times}")
    
    # Data access volume
    if behavior.data_access_volume > 1000:
        score += 20
        factors.append(f"High data access volume: {behavior.data_access_volume}")
    
    # Permission changes
    if behavior.permission_changes > 3:
        score += 25
        factors.append(f"Multiple permission changes: {behavior.permission_changes}")
    
    # Determine risk level
    if score >= 70:
        risk_level = "high"
        recommendations = [
            "Immediately review account activity",
            "Require multi-factor authentication",
            "Temporary access restrictions recommended",
            "Contact user to verify recent activity"
        ]
    elif score >= 40:
        risk_level = "medium"
        recommendations = [
            "Monitor account closely",
            "Review recent access logs",
            "Consider security awareness training"
        ]
    else:
        risk_level = "low"
        recommendations = ["Continue normal monitoring"]
    
    return RiskScore(
        risk_level=risk_level,
        score=min(100, score),
        factors=factors,
        recommendations=recommendations
    )
```

### Burnout Detection

```python
# ml-service/main.py (continued)
class WorkloadData(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_task_completion_time: float  # hours
    overtime_hours: float
    days_since_break: int
    task_complexity_score: float  # 1-10

class BurnoutAnalysis(BaseModel):
    burnout_risk: str  # low, medium, high
    score: float  # 0-100
    indicators: list[str]
    suggestions: list[str]

@app.post("/detect-burnout", response_model=BurnoutAnalysis)
async def detect_burnout(workload: WorkloadData):
    score = 0
    indicators = []
    
    # Task overload
    if workload.tasks_assigned > 20:
        score += 25
        indicators.append(f"Heavy task load: {workload.tasks_assigned} tasks")
    
    # Completion rate
    completion_rate = (workload.tasks_completed / max(1, workload.tasks_assigned)) * 100
    if completion_rate < 50:
        score += 20
        indicators.append(f"Low completion rate: {completion_rate:.1f}%")
    
    # Overtime
    if workload.overtime_hours > 20:
        score += 30
        indicators.append(f"Excessive overtime: {workload.overtime_hours} hours")
    
    # No breaks
    if workload.days_since_break > 30:
        score += 15
        indicators.append(f"No break in {workload.days_since_break} days")
    
    # High complexity
    if workload.task_complexity_score > 7:
        score += 10
        indicators.append(f"High task complexity: {workload.task_complexity_score}/10")
    
    # Determine risk
    if score >= 70:
        burnout_risk = "high"
        suggestions = [
            "Immediate workload reduction recommended",
            "Schedule mandatory break/vacation",
            "Redistribute tasks to team members",
            "One-on-one meeting with manager required"
        ]
    elif score >= 40:
        burnout_risk = "medium"
        suggestions = [
            "Monitor workload closely",
            "Consider task redistribution",
            "Encourage work-life balance",
            "Schedule check-in meeting"
        ]
    else:
        burnout_risk = "low"
        suggestions = ["Maintain current work patterns"]
    
    return BurnoutAnalysis(
        burnout_risk=burnout_risk,
        score=min(100, score),
        indicators=indicators,
        suggestions=suggestions
    )
```

### Project Delay Prediction

```python
# ml-service/main.py (continued)
class ProjectMetrics(BaseModel):
    project_id: str
    total_tasks: int
    completed_tasks: int
    days_elapsed: int
    days_remaining: int
    avg_completion_rate: float  # tasks per day
    blocked_tasks: int
    team_size: int

class DelayPrediction(BaseModel):
    will_delay: bool
    predicted_delay_days: int
    confidence: float
    risk_factors: list[str]
    mitigation_strategies: list[str]

@app.post("/predict-delay", response_model=DelayPrediction)
async def predict_project_delay(metrics: ProjectMetrics):
    # Calculate completion velocity
    current_velocity = metrics.avg_completion_rate
    required_velocity = (metrics.total_tasks - metrics.completed_tasks) / max(1, metrics.days_remaining)
    
    risk_score = 0
    risk_factors = []
    
    # Velocity check
    if current_velocity < required_velocity:
        velocity_gap = ((required_velocity - current_velocity) / required_velocity) * 100
        risk_score += min(40, velocity_gap)
        risk_factors.append(f"Current velocity ({current_velocity:.1f}) below required ({required_velocity:.1f})")
    
    # Blocked tasks
    if metrics.blocked_tasks > 0:
        block_percentage = (metrics.blocked_tasks / max(1, metrics.total_tasks)) * 100
        risk_score += min(30, block_percentage * 2)
        risk_factors.append(f"{metrics.blocked_tasks} blocked tasks ({block_percentage:.1f}%)")
    
    # Completion rate
    completion_percentage = (metrics.completed_tasks / max(1, metrics.total_tasks)) * 100
    time_percentage = (metrics.days_elapsed / max(1, metrics.days_elapsed + metrics.days_remaining)) * 100
    
    if completion_percentage < time_percentage - 10:
        risk_score += 30
        risk_factors.append(f"Behind schedule: {completion_percentage:.1f}% done vs {time_percentage:.1f}% time elapsed")
    
    # Predict delay
    will_delay = risk_score > 50
    predicted_delay_days = 0
    
    if will_delay:
        remaining_tasks = metrics.total_tasks - metrics.completed_tasks
        predicted_days_needed = remaining_tasks / max(0.1, current_velocity)
        predicted_delay_days = max(0, int(predicted_days_needed - metrics.days_remaining))
    
    # Mitigation strategies
    strategies = []
    if will_delay:
        if metrics.blocked_tasks > 0:
            strategies.append("Prioritize unblocking tasks immediately")
        if current_velocity < required_velocity:
            strategies.append(f"Increase team velocity by {((required_velocity/current_velocity - 1) * 100):.0f}%")
        if metrics.team_size < 5:
            strategies.append("Consider adding team members")
        strategies.append("Re-evaluate scope and prioritize critical features")
        strategies.append("Implement daily standups to address blockers")
    
    confidence = min(0.95, 0.5 + (len(risk_factors) * 0.15))
    
    return DelayPrediction(
        will_delay=will_delay,
        predicted_delay_days=predicted_delay_days,
        confidence=confidence,
        risk_factors=risk_factors,
        mitigation_strategies=strategies if strategies else ["Project on track"]
    )
```

## Frontend Integration

### Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const useAuth = () => useContext(AuthContext);

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [token, setToken] = useState(localStorage.getItem('token'));

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
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(res.data.data);
    } catch (error) {
      console.error('Auth error:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    
    const { token, user } = res.data;
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
    <AuthContext.Provider value={{ user, loading, login, logout, isAuthenticated: !!user }}>
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

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks/my-tasks`);
      const tasksByStatus = {
        todo: res.data.data.filter(t => t.status === 'todo'),
        inprogress: res.data.data.filter(t => t.status === 'inprogress'),
        done: res.data.data.filter(t => t.status === 'done')
      };
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks(); // Refresh tasks
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const TaskCard = ({ task }) => (
    <div
      className="task-card"
      draggable
      onDragStart={(e) => handleDragStart(e, task._id)}
    >
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        {task.timeSpent && <span>⏱ {Math.floor(task.timeSpent / 60)}m</span>}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      <div
        className="kanban-column"
        onDrop={(e) => handleDrop(e, 'todo')}
        onDragOver={handleDragOver}
      >
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      
      <div
        className="kanban-column"
        onDrop={(e) => handleDrop(e, 'inprogress')}
        onDragOver={handleDragOver}
      >
        <h3>In Progress ({tasks.inprogress.length})</h3>
        {tasks.inprogress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      
      <div
        className="kanban-column"
        onDrop={(e) => handleDrop(e, 'done')}
        onDragOver={handleDragOver}
      >
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
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
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    risk: null,
    burnout: null,
    loading: true
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Get user behavior data from backend
      const behaviorRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/analytics/user/${userId}/behavior`
      );
      
      // Get ML predictions
      const [riskRes, burnoutRes] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_URL}/predict-risk`, behaviorRes.data),
        axios.post(`${process.env.REACT_APP_ML_URL}/detect-burnout`, behaviorRes.data.workload)
      ]);
      
      setAnalytics({
        risk: riskRes.data,
        burnout: burnoutRes.data,
        loading: false
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
      setAnalytics(prev => ({ ...prev, loading: false }));
    }
  };

  if (analytics.loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="analytics-card">
        <h3>Security Risk Analysis</h3>
        <div className={`risk-badge ${analytics.risk?.risk_level}`}>
          {analytics.risk?.risk_level?.toUpperCase()} RISK
        </div>
        <p>Score: {analytics.risk?.score?.toFixed(1)}/100</p>
        
        <h4>Risk Factors:</h4>
        <ul>
          {analytics.risk?.factors?.map((factor, idx) => (
            <li key={idx}>{factor}</li>
          ))}
        </ul>
        
        <h4>Recommendations:</h4>
        <ul>
          {analytics.risk?.recommendations?.map((rec, idx) => (
            <li key={idx}>{rec}</li>
          ))}
        </ul>
      </div>
      
      <div className="analytics-card">
        <h3>Burnout Detection</h3>
        <div className={`risk-badge ${analytics.burnout?.burnout_risk}`}>
          {analytics.burnout?.burnout_risk?.toUpperCase()} RISK
        </div>
        <p>Score: {analytics.burnout?.score?.toFixed(1)}/100</p>
        
        <h4>Indicators:</h4>
        <ul>
          {analytics.burnout?.indicators?.map((indicator, idx) => (
            <li key={idx}>{
