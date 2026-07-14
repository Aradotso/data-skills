---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and predictive insights
triggers:
  - "build a user management system with AI analytics"
  - "create an enterprise user dashboard with task tracking"
  - "implement AI-based risk detection for users"
  - "set up a kanban board with time tracking"
  - "integrate ML models for burnout detection"
  - "build a ticket management system with AI classification"
  - "create role-based access control with JWT authentication"
  - "implement anomaly detection for user behavior"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack web application that combines user/task management with AI-powered insights. Built with React, Node.js, and FastAPI, it provides risk detection, anomaly detection, burnout analysis, and predictive project insights to improve organizational productivity and decision-making.

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance

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
EOF

npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
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
REACT_APP_ML_API_URL=http://localhost:8000
EOF

npm start
```

## Project Architecture

```
├── frontend/          # React application
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── services/
│   │   └── utils/
├── backend/           # Node.js API server
│   ├── controllers/
│   ├── models/
│   ├── routes/
│   └── middleware/
└── ml-service/        # FastAPI ML microservice
    ├── models/
    ├── services/
    └── main.py
```

## Core Components

### 1. Authentication & Authorization

**Backend: User Registration**

```javascript
// backend/controllers/authController.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.register = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Create user
    const user = await User.create({
      name,
      email,
      password, // Should be hashed in model pre-save hook
      role: role || 'user'
    });
    
    // Generate JWT
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
};
```

**Frontend: Authentication Service**

```javascript
// frontend/src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

class AuthService {
  async login(email, password) {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    
    return response.data;
  }
  
  logout() {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  }
  
  getCurrentUser() {
    return JSON.parse(localStorage.getItem('user'));
  }
  
  getAuthHeader() {
    const token = localStorage.getItem('token');
    return token ? { Authorization: `Bearer ${token}` } : {};
  }
}

export default new AuthService();
```

**Middleware: Role-Based Access Control**

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization?.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ message: 'Not authorized' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Token invalid' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        message: `Role ${req.user.role} not authorized`
      });
    }
    next();
  };
};
```

### 2. Task Management with Kanban Board

**Backend: Task Model**

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
    enum: ['todo', 'inprogress', 'done'],
    default: 'todo'
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  timeTracked: {
    type: Number,
    default: 0 // in seconds
  },
  dueDate: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', taskSchema);
```

**Backend: Task Controller**

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      assignedBy: req.user._id
    });
    
    await task.populate('assignedTo assignedBy', 'name email');
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.getTasks = async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user._id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo assignedBy', 'name email')
      .sort('-createdAt');
    
    res.json({ success: true, data: tasks });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status },
      { new: true, runValidators: true }
    ).populate('assignedTo assignedBy', 'name email');
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.updateTimeTracked = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeTracked: req.body.seconds } },
      { new: true }
    );
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};
```

**Frontend: Kanban Board Component**

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import authService from '../services/authService';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks`, {
        headers: authService.getAuthHeader()
      });
      
      const grouped = {
        todo: response.data.data.filter(t => t.status === 'todo'),
        inprogress: response.data.data.filter(t => t.status === 'inprogress'),
        done: response.data.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const handleDragStart = (e, task) => {
    e.dataTransfer.setData('taskId', task._id);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    
    try {
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: authService.getAuthHeader() }
      );
      
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const Column = ({ status, title, tasks }) => (
    <div
      className="kanban-column"
      onDragOver={(e) => e.preventDefault()}
      onDrop={(e) => handleDrop(e, status)}
    >
      <h3>{title} ({tasks.length})</h3>
      {tasks.map(task => (
        <div
          key={task._id}
          className="task-card"
          draggable
          onDragStart={(e) => handleDragStart(e, task)}
        >
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority ${task.priority}`}>{task.priority}</span>
          <div className="time-tracked">
            ⏱️ {Math.floor(task.timeTracked / 60)}m
          </div>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      <Column status="todo" title="To Do" tasks={tasks.todo} />
      <Column status="inprogress" title="In Progress" tasks={tasks.inprogress} />
      <Column status="done" title="Done" tasks={tasks.done} />
    </div>
  );
};

export default KanbanBoard;
```

### 3. Time Tracking

**Frontend: Time Tracker Component**

```javascript
// frontend/src/components/TimeTracker.jsx
import React, { useState, useEffect, useRef } from 'react';
import axios from 'axios';
import authService from '../services/authService';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const intervalRef = useRef(null);
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    };
  }, []);

  const startTimer = () => {
    setIsRunning(true);
    intervalRef.current = setInterval(() => {
      setElapsedTime(prev => prev + 1);
    }, 1000);
  };

  const stopTimer = async () => {
    setIsRunning(false);
    clearInterval(intervalRef.current);
    
    if (elapsedTime > 0) {
      try {
        await axios.patch(
          `${API_URL}/api/tasks/${taskId}/time`,
          { seconds: elapsedTime },
          { headers: authService.getAuthHeader() }
        );
        setElapsedTime(0);
      } catch (error) {
        console.error('Error saving time:', error);
      }
    }
  };

  const formatTime = (seconds) => {
    const h = Math.floor(seconds / 3600);
    const m = Math.floor((seconds % 3600) / 60);
    const s = seconds % 60;
    return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(elapsedTime)}</div>
      <button onClick={isRunning ? stopTimer : startTimer}>
        {isRunning ? 'Stop' : 'Start'}
      </button>
    </div>
  );
};

export default TimeTracker;
```

### 4. Support Ticket System

**Backend: Ticket Model**

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
    enum: ['technical', 'billing', 'general', 'urgent'],
    default: 'general'
  },
  status: {
    type: String,
    enum: ['open', 'in_progress', 'resolved', 'closed'],
    default: 'open'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical']
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
  aiClassified: {
    type: Boolean,
    default: false
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

**Backend: Ticket Controller with AI Integration**

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

exports.createTicket = async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Create initial ticket
    const ticket = await Ticket.create({
      title,
      description,
      createdBy: req.user._id
    });
    
    // Call ML service for AI classification
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/classify-ticket`,
        { title, description }
      );
      
      ticket.category = mlResponse.data.category;
      ticket.priority = mlResponse.data.priority;
      ticket.aiClassified = true;
      await ticket.save();
    } catch (mlError) {
      console.error('ML classification failed:', mlError);
    }
    
    await ticket.populate('createdBy', 'name email');
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.getTickets = async (req, res) => {
  try {
    const query = req.user.role === 'admin'
      ? {}
      : { createdBy: req.user._id };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy assignedTo', 'name email')
      .sort('-createdAt');
    
    res.json({ success: true, data: tickets });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### 5. AI/ML Services

**ML Service: Main API**

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib
import os

app = FastAPI()

# Models storage
models_path = os.getenv('MODEL_PATH', './models')
os.makedirs(models_path, exist_ok=True)

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskAnalysisRequest(BaseModel):
    user_id: str
    task_completion_rate: float
    average_delay: float
    ticket_count: int
    overtime_hours: float

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    login_time: str
    location: str
    device: str
    actions_count: int

@app.post("/api/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """
    Classify support tickets into categories and assign priority
    """
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple rule-based classification (replace with trained model)
        category = "general"
        priority = "medium"
        
        # Category classification
        if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = "technical"
        elif any(word in text for word in ['payment', 'invoice', 'charge']):
            category = "billing"
        elif any(word in text for word in ['urgent', 'critical', 'immediately']):
            category = "urgent"
        
        # Priority assignment
        if category == "urgent" or "critical" in text:
            priority = "critical"
        elif "important" in text or category == "technical":
            priority = "high"
        elif "low priority" in text:
            priority = "low"
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/risk-analysis")
async def analyze_risk(request: RiskAnalysisRequest):
    """
    Analyze user risk based on behavior patterns
    """
    try:
        # Calculate risk score (0-100)
        risk_score = 0
        
        # Low completion rate increases risk
        if request.task_completion_rate < 0.5:
            risk_score += 30
        elif request.task_completion_rate < 0.7:
            risk_score += 15
        
        # High average delay increases risk
        if request.average_delay > 5:
            risk_score += 25
        elif request.average_delay > 2:
            risk_score += 10
        
        # High ticket count indicates issues
        if request.ticket_count > 10:
            risk_score += 20
        elif request.ticket_count > 5:
            risk_score += 10
        
        # Excessive overtime may indicate burnout
        if request.overtime_hours > 20:
            risk_score += 25
        elif request.overtime_hours > 10:
            risk_score += 15
        
        risk_level = "low"
        if risk_score > 60:
            risk_level = "high"
        elif risk_score > 30:
            risk_level = "medium"
        
        return {
            "user_id": request.user_id,
            "risk_score": min(risk_score, 100),
            "risk_level": risk_level,
            "recommendations": generate_recommendations(risk_score, request)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def generate_recommendations(risk_score, data):
    recommendations = []
    
    if data.task_completion_rate < 0.7:
        recommendations.append("Consider redistributing tasks or providing additional support")
    
    if data.average_delay > 2:
        recommendations.append("Review task complexity and deadlines")
    
    if data.overtime_hours > 10:
        recommendations.append("Monitor workload to prevent burnout")
    
    if data.ticket_count > 5:
        recommendations.append("Address recurring issues through training or resources")
    
    return recommendations

@app.post("/api/anomaly-detection")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """
    Detect anomalous user behavior for security
    """
    try:
        anomaly_score = 0
        flags = []
        
        # Check login time (simple heuristic)
        hour = int(request.login_time.split(':')[0])
        if hour < 6 or hour > 22:
            anomaly_score += 20
            flags.append("Unusual login time")
        
        # High action count in short period
        if request.actions_count > 100:
            anomaly_score += 30
            flags.append("Unusually high activity")
        
        is_anomaly = anomaly_score > 30
        
        return {
            "user_id": request.user_id,
            "is_anomaly": is_anomaly,
            "anomaly_score": anomaly_score,
            "flags": flags,
            "severity": "high" if anomaly_score > 50 else "medium" if anomaly_score > 30 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/burnout-detection")
async def detect_burnout(request: dict):
    """
    Detect employee burnout based on work patterns
    """
    try:
        burnout_score = 0
        
        # Factors contributing to burnout
        overtime = request.get('overtime_hours', 0)
        task_load = request.get('tasks_assigned', 0)
        completion_rate = request.get('completion_rate', 1.0)
        consecutive_work_days = request.get('consecutive_work_days', 0)
        
        if overtime > 15:
            burnout_score += 30
        if task_load > 20:
            burnout_score += 25
        if completion_rate < 0.6:
            burnout_score += 20
        if consecutive_work_days > 10:
            burnout_score += 25
        
        burnout_level = "low"
        if burnout_score > 60:
            burnout_level = "high"
        elif burnout_score > 35:
            burnout_level = "medium"
        
        return {
            "user_id": request.get('user_id'),
            "burnout_score": min(burnout_score, 100),
            "burnout_level": burnout_level,
            "recommendations": [
                "Schedule time off" if burnout_score > 60 else None,
                "Reduce task load" if task_load > 20 else None,
                "Limit overtime hours" if overtime > 15 else None
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

**Backend: Integration with ML Service**

```javascript
// backend/services/mlService.js
const axios = require('axios');

const ML_API = process.env.ML_SERVICE_URL || 'http://localhost:8000';

class MLService {
  async analyzeUserRisk(userId, userData) {
    try {
      const response = await axios.post(`${ML_API}/api/risk-analysis`, {
        user_id: userId,
        task_completion_rate: userData.completionRate,
        average_delay: userData.avgDelay,
        ticket_count: userData.ticketCount,
        overtime_hours: userData.overtimeHours
      });
      
      return response.data;
    } catch (error) {
      console.error('ML Risk Analysis Error:', error);
      throw error;
    }
  }
  
  async detectAnomalies(userId, activityData) {
    try {
      const response = await axios.post(`${ML_API}/api/anomaly-detection`, {
        user_id: userId,
        login_time: activityData.loginTime,
        location: activityData.location,
        device: activityData.device,
        actions_count: activityData.actionsCount
      });
      
      return response.data;
    } catch (error) {
      console.error('ML Anomaly Detection Error:', error);
      throw error;
    }
  }
  
  async detectBurnout(userId, workData) {
    try {
      const response = await axios.post(`${ML_API}/api/burnout-detection`, {
        user_id: userId,
        overtime_hours: workData.overtimeHours,
        tasks_assigned: workData.tasksAssigned,
        completion_rate: workData.completionRate,
        consecutive_work_days: workData.consecutiveWorkDays
      });
      
      return response.data;
    } catch (error) {
      console.error('ML Burnout Detection Error:', error);
      throw error;
    }
  }
}

module.exports = new MLService();
```

### 6. Admin Dashboard with Analytics

**Backend: Analytics Controller**

```javascript
// backend/controllers/analyticsController.js
const User = require('../models/User');
const Task = require('../models/Task');
const Ticket = require('../models/Ticket');
const mlService = require('../services/mlService');

exports.getDashboardAnalytics = async (req, res) => {
  try {
    // Basic statistics
    const totalUsers = await User.countDocuments({ role: 'user' });
    const totalTasks = await Task.countDocuments();
    const totalTickets = await Ticket.countDocuments();
    
    // Task statistics by status
    const tasksByStatus = await Task.aggregate([
      { $group: { _id: '$status', count: { $sum: 1 } } }
    ]);
    
    // Ticket statistics by category
    const ticketsByCategory = await Ticket.aggregate([
      { $group: { _id: '$category', count: { $sum: 1 } } }
    ]);
    
    // User performance metrics
    const userMetrics = await Task.aggregate([
      {
        $group: {
          _id: '$assignedTo',
          totalTasks: { $sum: 1 },
          completedTasks: {
            $sum: { $cond: [{ $eq: ['$status', 'done'] }, 1, 0] }
          },
          totalTimeTracked: { $sum: '$timeTracked' }
        }
      },
      {
        $lookup: {
          from: 'users',
          localField: '_id',
          foreignField: '_id',
          as: 'user'
        }
      },
      { $unwind: '$user' },
      {
        $project: {
          userId: '$_id',
          name: '$user.name',
          email: '$user.email',
          totalTasks: 1,
          completedTasks: 1,
          completionRate: {
            $cond: [
              { $eq: ['$totalTasks', 0] },
              0,
              { $divide: ['$completedTasks', '$totalTasks'] }
            ]
          },
          avgTimePerTask: {
            $cond: [
              { $eq: ['$totalTasks', 0] },
              0,
              { $divide: ['$totalTimeTracked', '$totalTasks'] }
            ]
          }
        }
      }
    ]);
    
    res.json({
      success: true,
      data: {
        overview: {
          totalUsers,
          totalTasks,
          totalTickets
        },
        tasksByStatus: tasksByStatus.reduce((acc, curr) => {
          acc[curr._id] = curr.count;
          return acc;
        }, {}),
        ticketsByCategory: ticketsByCategory.reduce((acc, curr) => {
          acc[curr._id] = curr.count;
          return acc;
        }, {}),
        userMetrics
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.getUserRiskAnalysis = async (req, res) => {
  try {
    const userId = req.params.userId;
    
    // Gather user data
    const tasks = await Task.find({ assignedTo: userId });
    const tickets = await Ticket.find({ createdBy: userId });
    
    const completedTasks = tasks.filter(t => t.status === 'done').length;
    const totalTasks = tasks.length;
    const completionRate = totalTasks > 0 ? completedTasks / totalTasks : 0;
    
    // Calculate average delay (simplified)
    const now = new Date();
    const delayedTasks = tasks.filter(t => 
      t.dueDate && new Date(t.dueDate) < now && t.status !== 'done'
    );
    const avgDelay = delayedTasks.length;
    
    // Get ML risk analysis
    const riskAnalysis = await mlService.analyzeUserRisk(userId, {
      completionRate,
      avgDelay,
      ticketCount: tickets.length,
      overtimeHours: 0
