---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and workforce insights
triggers:
  - "help me set up the enterprise user management system"
  - "how do I implement AI analytics for user behavior"
  - "configure JWT authentication for user management"
  - "integrate the ML service for risk detection"
  - "create a kanban board with task tracking"
  - "set up ticket classification with AI"
  - "implement burnout detection for users"
  - "build an admin dashboard with analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with machine learning capabilities. It provides AI-driven insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics. Built with React frontend, Node.js backend, FastAPI ML service, and MongoDB database.

## Installation

### Prerequisites

- Node.js 14+
- Python 3.8+
- MongoDB
- npm or yarn

### Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
npm start
# Runs on http://localhost:5000

# ML Service setup
cd ../ml-service
pip install -r requirements.txt
uvicorn main:app --reload
# Runs on http://localhost:8000

# Frontend setup
cd ../frontend
npm install
npm start
# Runs on http://localhost:3000
```

## Configuration

### Backend Environment Variables

Create `backend/.env`:

```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=24h
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

### ML Service Configuration

Create `ml-service/.env`:

```python
# ml-service/.env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

### Frontend Configuration

Create `frontend/.env`:

```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

## Backend API

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// User registration
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    // Check if user exists
    let user = await User.findOne({ email });
    if (user) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    // Create user
    user = new User({
      name,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate JWT
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ token, user: { id: user._id, name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// User login
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
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ token, user: { id: user._id, name: user.name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const auth = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const adminAuth = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { auth, adminAuth };
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { auth } = require('../middleware/auth');

// Get user tasks
router.get('/', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'name email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Create task
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, status, priority, assignedTo, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      status: status || 'todo',
      priority: priority || 'medium',
      assignedTo,
      assignedBy: req.user.id,
      dueDate
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Track time
router.post('/:id/time', auth, async (req, res) => {
  try {
    const { duration } = req.body; // duration in minutes
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.timeSpent = (task.timeSpent || 0) + duration;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Ticket Management

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { auth } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { subject, description, priority } = req.body;
    
    // Call ML service for classification
    let category = 'general';
    let aiPriority = priority || 'medium';
    
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        subject,
        description
      });
      category = mlResponse.data.category;
      aiPriority = mlResponse.data.priority;
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }
    
    const ticket = new Ticket({
      subject,
      description,
      category,
      priority: aiPriority,
      status: 'open',
      createdBy: req.user.id
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Get tickets
router.get('/', auth, async (req, res) => {
  try {
    const query = req.user.role === 'admin' ? {} : { createdBy: req.user.id };
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

## ML Service API

### FastAPI ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, tree
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
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)
risk_model = None

# Load models on startup
@app.on_event("startup")
async def load_models():
    global ticket_classifier, risk_model
    model_path = os.getenv("MODEL_PATH", "./models")
    
    try:
        ticket_classifier = joblib.load(f"{model_path}/ticket_classifier.pkl")
    except:
        print("Ticket classifier not found, using default")
    
    try:
        risk_model = joblib.load(f"{model_path}/risk_model.pkl")
    except:
        print("Risk model not found, using default")

# Request models
class TicketRequest(BaseModel):
    subject: str
    description: str

class UserBehaviorRequest(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_overdue: int
    avg_completion_time: float
    login_frequency: int
    support_tickets: int

class BurnoutRequest(BaseModel):
    user_id: str
    total_tasks: int
    active_tasks: int
    avg_hours_per_day: float
    overdue_tasks: int
    stress_indicators: List[float]

# Ticket classification
@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketRequest):
    try:
        # Simple rule-based classification (replace with trained model)
        text = f"{ticket.subject} {ticket.description}".lower()
        
        if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
            priority = 'high'
        elif any(word in text for word in ['access', 'password', 'login', 'permission']):
            category = 'access'
            priority = 'medium'
        elif any(word in text for word in ['feature', 'request', 'suggestion']):
            category = 'feature_request'
            priority = 'low'
        elif any(word in text for word in ['urgent', 'critical', 'asap']):
            category = 'general'
            priority = 'high'
        else:
            category = 'general'
            priority = 'medium'
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Risk detection
@app.post("/detect-risk")
async def detect_risk(behavior: UserBehaviorRequest):
    try:
        # Calculate risk score based on behavior patterns
        risk_score = 0.0
        risk_factors = []
        
        # High number of overdue tasks
        if behavior.tasks_overdue > 5:
            risk_score += 0.3
            risk_factors.append("High overdue task count")
        
        # Low completion rate
        completion_rate = behavior.tasks_completed / (behavior.tasks_completed + behavior.tasks_overdue + 0.1)
        if completion_rate < 0.5:
            risk_score += 0.25
            risk_factors.append("Low task completion rate")
        
        # Slow completion time
        if behavior.avg_completion_time > 7.0:  # days
            risk_score += 0.2
            risk_factors.append("Slow task completion")
        
        # Low login frequency
        if behavior.login_frequency < 3:  # per week
            risk_score += 0.15
            risk_factors.append("Low engagement")
        
        # High support tickets
        if behavior.support_tickets > 10:
            risk_score += 0.1
            risk_factors.append("High support ticket volume")
        
        risk_level = "low" if risk_score < 0.3 else "medium" if risk_score < 0.6 else "high"
        
        return {
            "user_id": behavior.user_id,
            "risk_score": min(risk_score, 1.0),
            "risk_level": risk_level,
            "risk_factors": risk_factors,
            "recommendation": get_risk_recommendation(risk_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_risk_recommendation(risk_level: str) -> str:
    recommendations = {
        "low": "Continue monitoring. User performance is stable.",
        "medium": "Consider check-in meeting. Review workload and provide support.",
        "high": "Immediate action required. Reassign tasks and provide direct assistance."
    }
    return recommendations.get(risk_level, "Monitor user activity")

# Anomaly detection
@app.post("/detect-anomaly")
async def detect_anomaly(behavior: UserBehaviorRequest):
    try:
        # Create feature vector
        features = {
            'tasks_completed': behavior.tasks_completed,
            'tasks_overdue': behavior.tasks_overdue,
            'avg_completion_time': behavior.avg_completion_time,
            'login_frequency': behavior.login_frequency,
            'support_tickets': behavior.support_tickets
        }
        
        # Update and score with anomaly detector
        anomaly_detector.learn_one(features)
        anomaly_score = anomaly_detector.score_one(features)
        
        is_anomaly = anomaly_score > 0.7
        
        return {
            "user_id": behavior.user_id,
            "anomaly_score": float(anomaly_score),
            "is_anomaly": is_anomaly,
            "severity": "high" if anomaly_score > 0.85 else "medium" if anomaly_score > 0.7 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout detection
@app.post("/detect-burnout")
async def detect_burnout(data: BurnoutRequest):
    try:
        burnout_score = 0.0
        indicators = []
        
        # High workload
        if data.active_tasks > 15:
            burnout_score += 0.25
            indicators.append("Excessive active tasks")
        
        # Long working hours
        if data.avg_hours_per_day > 9.0:
            burnout_score += 0.3
            indicators.append("Extended working hours")
        
        # High overdue rate
        overdue_rate = data.overdue_tasks / (data.total_tasks + 0.1)
        if overdue_rate > 0.3:
            burnout_score += 0.25
            indicators.append("High overdue task rate")
        
        # Stress indicators average
        if data.stress_indicators:
            avg_stress = np.mean(data.stress_indicators)
            if avg_stress > 0.7:
                burnout_score += 0.2
                indicators.append("Elevated stress levels")
        
        burnout_level = "low" if burnout_score < 0.4 else "medium" if burnout_score < 0.7 else "high"
        
        return {
            "user_id": data.user_id,
            "burnout_score": min(burnout_score, 1.0),
            "burnout_level": burnout_level,
            "indicators": indicators,
            "recommendation": get_burnout_recommendation(burnout_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_burnout_recommendation(burnout_level: str) -> str:
    recommendations = {
        "low": "User is managing workload well. Continue regular check-ins.",
        "medium": "Consider workload redistribution. Schedule wellness check-in.",
        "high": "Urgent intervention needed. Reduce workload immediately and provide mental health resources."
    }
    return recommendations.get(burnout_level, "Monitor user wellness")

# Health check
@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Frontend Components

### User Dashboard with Kanban Board

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './UserDashboard.css';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);
  const [activeTimer, setActiveTimer] = useState(null);
  const [timerSeconds, setTimerSeconds] = useState(0);

  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, []);

  useEffect(() => {
    let interval = null;
    if (activeTimer) {
      interval = setInterval(() => {
        setTimerSeconds(seconds => seconds + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [activeTimer]);

  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(`${API_URL}/api/tasks`, {
        headers: { Authorization: `Bearer ${token}` }
      });

      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done')
      };

      setTasks(categorized);
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
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const startTimer = (taskId) => {
    setActiveTimer(taskId);
    setTimerSeconds(0);
  };

  const stopTimer = async (taskId) => {
    try {
      const token = localStorage.getItem('token');
      const duration = Math.floor(timerSeconds / 60); // Convert to minutes
      
      await axios.post(
        `${API_URL}/api/tasks/${taskId}/time`,
        { duration },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      setActiveTimer(null);
      setTimerSeconds(0);
      fetchTasks();
    } catch (error) {
      console.error('Error tracking time:', error);
    }
  };

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const TaskCard = ({ task }) => (
    <div className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        {task.timeSpent && <span>⏱️ {task.timeSpent}min</span>}
      </div>
      
      {activeTimer === task._id ? (
        <div className="timer-active">
          <span>{formatTime(timerSeconds)}</span>
          <button onClick={() => stopTimer(task._id)}>Stop</button>
        </div>
      ) : (
        <button onClick={() => startTimer(task._id)} className="timer-btn">
          Start Timer
        </button>
      )}
      
      <div className="task-actions">
        {task.status === 'todo' && (
          <button onClick={() => updateTaskStatus(task._id, 'in_progress')}>
            Start
          </button>
        )}
        {task.status === 'in_progress' && (
          <>
            <button onClick={() => updateTaskStatus(task._id, 'todo')}>
              Back
            </button>
            <button onClick={() => updateTaskStatus(task._id, 'done')}>
              Complete
            </button>
          </>
        )}
      </div>
    </div>
  );

  if (loading) return <div>Loading...</div>;

  return (
    <div className="user-dashboard">
      <h2>My Tasks</h2>
      <div className="kanban-board">
        <div className="kanban-column">
          <h3>To Do ({tasks.todo.length})</h3>
          {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
        </div>
        
        <div className="kanban-column">
          <h3>In Progress ({tasks.inProgress.length})</h3>
          {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
        </div>
        
        <div className="kanban-column">
          <h3>Done ({tasks.done.length})</h3>
          {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './AdminDashboard.css';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);
  const [loading, setLoading] = useState(true);

  const API_URL = process.env.REACT_APP_API_URL;
  const ML_URL = process.env.REACT_APP_ML_SERVICE_URL;

  useEffect(() => {
    fetchAnalytics();
    fetchRiskAnalysis();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(`${API_URL}/api/admin/analytics`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      setAnalytics(response.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const fetchRiskAnalysis = async () => {
    try {
      const token = localStorage.getItem('token');
      const usersResponse = await axios.get(`${API_URL}/api/admin/users`, {
        headers: { Authorization: `Bearer ${token}` }
      });

      const riskPromises = usersResponse.data.map(async (user) => {
        try {
          const riskResponse = await axios.post(`${ML_URL}/detect-risk`, {
            user_id: user._id,
            tasks_completed: user.tasksCompleted || 0,
            tasks_overdue: user.tasksOverdue || 0,
            avg_completion_time: user.avgCompletionTime || 5,
            login_frequency: user.loginFrequency || 5,
            support_tickets: user.supportTickets || 0
          });
          return { ...user, risk: riskResponse.data };
        } catch (error) {
          return { ...user, risk: null };
        }
      });

      const usersWithRisk = await Promise.all(riskPromises);
      const highRisk = usersWithRisk.filter(u => u.risk?.risk_level === 'high');
      
      setRiskUsers(highRisk);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching risk analysis:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="admin-dashboard">
      <h2>Admin Analytics Dashboard</h2>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-value">{analytics?.totalUsers || 0}</p>
        </div>
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p className="stat-value">{analytics?.activeTasks || 0}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p className="stat-value">{analytics?.openTickets || 0}</p>
        </div>
        <div className="stat-card">
          <h3>Completion Rate</h3>
          <p className="stat-value">{analytics?.completionRate || 0}%</p>
        </div>
      </div>

      <div className="risk-section">
        <h3>High Risk Users 🚨</h3>
        {riskUsers.length === 0 ? (
          <p>No high-risk users detected</p>
        ) : (
          <table className="risk-table">
            <thead>
              <tr>
                <th>User</th>
                <th>Risk Score</th>
                <th>Factors</th>
                <th>Recommendation</th>
              </tr>
            </thead>
            <tbody>
              {riskUsers.map(user => (
                <tr key={user._id}>
                  <td>{user.name}</td>
                  <td>
                    <span className="risk-badge high">
                      {(user.risk.risk_score * 100).toFixed(0)}%
                    </span>
                  </td>
                  <td>
                    <ul className="risk-factors">
                      {user.risk.risk_factors.map((factor, i) => (
                        <li key={i}>{factor}</li>
                      ))}
                    </ul>
                  </td>
                  <td>{user.risk.recommendation}</td>
                </tr>
              ))}
            </tbody>
          </table>
        )}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

### Support Ticket Creation with AI

```javascript
// frontend/src/components/CreateTicket.jsx
import React, { useState } from 'react';
import axios from 'axios';

const CreateTicket = ({ onSuccess }) => {
  const [formData, setFormData] = useState({
    subject: '',
    description: '',
    priority: 'medium'
  });
  const [submitting, setSubmitting] = useState(false);
  const [aiPrediction, setAiPrediction] = useState(null);

  const API_URL = process.env.REACT_APP_API_URL;

  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const predictCategory = async () => {
    if (formData.subject && formData.description) {
      try {
        const response = await axios.post(
          `${process.env.REACT_APP_ML_SERVICE_URL}/classify-ticket`,
          {
            subject: formData.subject,
            description: formData.description
          }
        );
        setAiPrediction(response.data);
      } catch (error) {
        console.error('Prediction error:', error);
      }
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setSubmitting(true);

    try {
      const token = localStorage.getItem('token');
      await axios.post(
        `${API_URL}/api/tickets`,
        formData,
        { headers:
