---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task tracking using React, Node.js, MongoDB, and FastAPI ML service
triggers:
  - set up enterprise user management system with AI analytics
  - integrate AI-powered user management and task tracking
  - build user management system with anomaly detection
  - implement AI analytics for employee burnout and risk prediction
  - create admin dashboard with user and task management
  - add AI ticket classification and routing system
  - configure kanban board with time tracking features
  - deploy user management system with ML service
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System, a full-stack application that combines traditional user/task management with AI-powered analytics for risk detection, burnout analysis, anomaly detection, and predictive insights. The system uses React for frontend, Node.js for backend, MongoDB for data persistence, and FastAPI with scikit-learn for ML services.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control, user profiles, authentication with JWT
- **Task Management**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: Smart ticket routing and classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization-wide analytics, audit logs, security alerts
- **User Dashboard**: Personal task overview, performance insights, notifications

## Installation

### Prerequisites
- Node.js 14+
- Python 3.8+
- MongoDB instance
- npm or yarn

### Clone and Setup

```bash
# Clone the repository
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
MONGO_URI=${MONGODB_CONNECTION_STRING}
JWT_SECRET=${JWT_SECRET_KEY}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=${MONGODB_CONNECTION_STRING}
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

ML service runs at `http://localhost:8000`

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

Frontend runs at `http://localhost:3000`

## Project Structure

```
Enterprise-User-Management-System-with-AI-Analytics/
├── frontend/              # React application
│   ├── src/
│   │   ├── components/   # UI components
│   │   ├── pages/        # Dashboard, Login, etc.
│   │   ├── services/     # API calls
│   │   └── utils/        # Helper functions
├── backend/              # Node.js REST API
│   ├── models/          # MongoDB schemas
│   ├── routes/          # API routes
│   ├── controllers/     # Business logic
│   ├── middleware/      # Auth, validation
│   └── server.js        # Entry point
└── ml-service/          # FastAPI ML service
    ├── models/          # Trained models
    ├── routes/          # ML endpoints
    ├── services/        # ML logic
    └── main.py          # FastAPI app
```

## Backend API (Node.js)

### User Authentication

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
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
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
    
    // Generate JWT
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
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
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ token, user: { id: user._id, name: user.name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Auth Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No token, authorization denied' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Token is not valid' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Task Management

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: { type: Date },
  timeSpent: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', TaskSchema);

// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

// Get user tasks
router.get('/my-tasks', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'name email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task (admin)
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.id,
      priority,
      dueDate
    });
    
    await task.save();
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
      { _id: req.params.id, assignedTo: req.user.id },
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update time spent
router.patch('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { $inc: { timeSpent: timeSpent }, updatedAt: Date.now() },
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
// backend/models/Ticket.js
const mongoose = require('mongoose');

const TicketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { type: String, enum: ['technical', 'hr', 'admin', 'other'], default: 'other' },
  priority: { type: String, enum: ['low', 'medium', 'high', 'critical'] },
  status: { type: String, enum: ['open', 'in-progress', 'resolved', 'closed'], default: 'open' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassified: { type: Boolean, default: false },
  aiPredictedCategory: { type: String },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', TicketSchema);

// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authMiddleware } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Call ML service for classification
    let aiCategory = null;
    let aiPriority = null;
    
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { title, description }
      );
      aiCategory = mlResponse.data.category;
      aiPriority = mlResponse.data.priority;
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      createdBy: req.user.id,
      category: aiCategory || 'other',
      priority: aiPriority || 'medium',
      aiClassified: !!aiCategory,
      aiPredictedCategory: aiCategory
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .populate('assignedTo', 'name email')
      .sort({ createdAt: -1 });
    res.json(tickets);
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
from typing import Optional, List
import numpy as np
from datetime import datetime
import pickle
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models directory
MODEL_DIR = os.getenv("MODEL_PATH", "./models")

# Request models
class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_login_attempts: int
    unusual_access_times: int
    data_access_frequency: int
    permission_changes: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    average_task_time: float
    overtime_hours: float
    days_since_last_break: int

class ProjectInsightRequest(BaseModel):
    project_id: str
    tasks_total: int
    tasks_completed: int
    tasks_in_progress: int
    average_completion_time: float
    days_until_deadline: int

@app.get("/")
def read_root():
    return {"service": "Enterprise User Management ML Service", "status": "active"}

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """AI-based ticket classification and routing"""
    try:
        # Simple keyword-based classification (replace with trained model)
        text = (request.title + " " + request.description).lower()
        
        # Category classification
        if any(word in text for word in ['password', 'login', 'access', 'system', 'error', 'bug']):
            category = 'technical'
        elif any(word in text for word in ['leave', 'salary', 'payroll', 'benefits', 'hr']):
            category = 'hr'
        elif any(word in text for word in ['policy', 'compliance', 'admin', 'document']):
            category = 'admin'
        else:
            category = 'other'
        
        # Priority classification
        if any(word in text for word in ['urgent', 'critical', 'emergency', 'asap', 'immediately']):
            priority = 'critical'
        elif any(word in text for word in ['high', 'important', 'soon']):
            priority = 'high'
        elif any(word in text for word in ['low', 'minor', 'whenever']):
            priority = 'low'
        else:
            priority = 'medium'
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Risk prediction based on user behavior"""
    try:
        # Calculate risk score (0-100)
        risk_score = 0
        
        # Failed login attempts (max 40 points)
        risk_score += min(request.failed_login_attempts * 10, 40)
        
        # Unusual access times (max 20 points)
        risk_score += min(request.unusual_access_times * 5, 20)
        
        # Data access frequency (max 20 points)
        if request.data_access_frequency > 100:
            risk_score += 20
        elif request.data_access_frequency > 50:
            risk_score += 10
        
        # Permission changes (max 20 points)
        risk_score += min(request.permission_changes * 10, 20)
        
        # Determine risk level
        if risk_score >= 70:
            risk_level = "critical"
        elif risk_score >= 50:
            risk_level = "high"
        elif risk_score >= 30:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return {
            "user_id": request.user_id,
            "risk_score": risk_score,
            "risk_level": risk_level,
            "recommendations": get_risk_recommendations(risk_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Burnout detection using workload analysis"""
    try:
        burnout_score = 0
        
        # High task completion rate (potential overwork)
        if request.tasks_completed > 30:
            burnout_score += 30
        elif request.tasks_completed > 20:
            burnout_score += 20
        
        # Average task time (faster completion may indicate stress)
        if request.average_task_time < 30:  # less than 30 minutes
            burnout_score += 20
        
        # Overtime hours
        burnout_score += min(request.overtime_hours * 2, 30)
        
        # Days since last break
        if request.days_since_last_break > 30:
            burnout_score += 20
        elif request.days_since_last_break > 15:
            burnout_score += 10
        
        # Determine burnout level
        if burnout_score >= 70:
            burnout_level = "critical"
        elif burnout_score >= 50:
            burnout_level = "high"
        elif burnout_score >= 30:
            burnout_level = "moderate"
        else:
            burnout_level = "low"
        
        return {
            "user_id": request.user_id,
            "burnout_score": burnout_score,
            "burnout_level": burnout_level,
            "recommendations": get_burnout_recommendations(burnout_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/project-insights")
async def project_insights(request: ProjectInsightRequest):
    """Predictive project insights (delay prediction)"""
    try:
        # Calculate completion percentage
        completion_rate = (request.tasks_completed / request.tasks_total * 100) if request.tasks_total > 0 else 0
        
        # Estimate remaining time
        if request.tasks_completed > 0:
            estimated_time_per_task = request.average_completion_time
            remaining_tasks = request.tasks_total - request.tasks_completed - request.tasks_in_progress
            estimated_days_needed = (remaining_tasks * estimated_time_per_task) / 8  # 8 hours per day
        else:
            estimated_days_needed = request.days_until_deadline
        
        # Predict delay
        delay_risk = "none"
        delay_days = 0
        
        if estimated_days_needed > request.days_until_deadline:
            delay_days = estimated_days_needed - request.days_until_deadline
            if delay_days > 7:
                delay_risk = "high"
            elif delay_days > 3:
                delay_risk = "medium"
            else:
                delay_risk = "low"
        
        return {
            "project_id": request.project_id,
            "completion_rate": round(completion_rate, 2),
            "estimated_days_needed": round(estimated_days_needed, 2),
            "days_until_deadline": request.days_until_deadline,
            "delay_risk": delay_risk,
            "predicted_delay_days": round(delay_days, 2),
            "recommendations": get_project_recommendations(delay_risk, completion_rate)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_risk_recommendations(risk_level: str) -> List[str]:
    recommendations = {
        "critical": [
            "Immediately suspend user access",
            "Conduct security audit",
            "Review all recent activities",
            "Contact security team"
        ],
        "high": [
            "Enable multi-factor authentication",
            "Review access permissions",
            "Monitor user activity closely"
        ],
        "medium": [
            "Send security awareness reminder",
            "Review recent login attempts"
        ],
        "low": [
            "Continue normal monitoring"
        ]
    }
    return recommendations.get(risk_level, [])

def get_burnout_recommendations(burnout_level: str) -> List[str]:
    recommendations = {
        "critical": [
            "Immediate intervention required",
            "Reduce workload significantly",
            "Mandate time off",
            "Schedule wellness check-in"
        ],
        "high": [
            "Redistribute tasks",
            "Encourage break time",
            "Consider workload adjustment"
        ],
        "moderate": [
            "Monitor workload",
            "Suggest flexible hours"
        ],
        "low": [
            "Maintain current balance"
        ]
    }
    return recommendations.get(burnout_level, [])

def get_project_recommendations(delay_risk: str, completion_rate: float) -> List[str]:
    recommendations = []
    
    if delay_risk == "high":
        recommendations.extend([
            "Add more resources to the project",
            "Re-prioritize tasks",
            "Consider extending deadline",
            "Schedule emergency team meeting"
        ])
    elif delay_risk == "medium":
        recommendations.extend([
            "Monitor progress daily",
            "Identify blockers",
            "Optimize task allocation"
        ])
    
    if completion_rate < 25:
        recommendations.append("Project is significantly behind schedule")
    
    return recommendations if recommendations else ["Project on track"]

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Requirements File

```python
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
numpy==1.24.3
scikit-learn==1.3.2
river==0.19.0
pandas==2.0.3
python-multipart==0.0.6
pymongo==4.6.0
python-dotenv==1.0.0
```

## Frontend (React)

### API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

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

// Auth services
export const authService = {
  login: (email, password) => api.post('/auth/login', { email, password }),
  register: (userData) => api.post('/auth/register', userData),
  logout: () => {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  },
};

// Task services
export const taskService = {
  getMyTasks: () => api.get('/tasks/my-tasks'),
  createTask: (taskData) => api.post('/tasks', taskData),
  updateTaskStatus: (taskId, status) => api.patch(`/tasks/${taskId}/status`, { status }),
  updateTaskTime: (taskId, timeSpent) => api.patch(`/tasks/${taskId}/time`, { timeSpent }),
  deleteTask: (taskId) => api.delete(`/tasks/${taskId}`),
};

// Ticket services
export const ticketService = {
  getMyTickets: () => api.get('/tickets/my-tickets'),
  createTicket: (ticketData) => api.post('/tickets', ticketData),
  getAllTickets: () => api.get('/tickets'),
  updateTicketStatus: (ticketId, status) => api.patch(`/tickets/${ticketId}`, { status }),
};

// ML services
export const mlService = {
  classifyTicket: (title, description) =>
    axios.post(`${ML_API_URL}/classify-ticket`, { title, description }),
  predictRisk: (userData) =>
    axios.post(`${ML_API_URL}/predict-risk`, userData),
  analyzeBurnout: (userData) =>
    axios.post(`${ML_API_URL}/analyze-burnout`, userData),
  getProjectInsights: (projectData) =>
    axios.post(`${ML_API_URL}/project-insights`, projectData),
};

// User services
export const userService = {
  getAllUsers: () => api.get('/users'),
  getUserById: (userId) => api.get(`/users/${userId}`),
  updateUser: (userId, userData) => api.patch(`/users/${userId}`, userData),
  deleteUser: (userId) => api.delete(`/users/${userId}`),
};

export default api;
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskService.getMyTasks();
      const groupedTasks = {
        todo: response.data.filter(t => t.status === 'todo'),
        'in-progress': response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done'),
      };
      setTasks(groupedTasks);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const handleDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');

    if (currentStatus === newStatus) return;

    try {
      await taskService.updateTaskStatus(taskId, newStatus);
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const formatTime = (minutes) => {
    const hours = Math.floor(minutes / 60);
    const mins = minutes % 60;
    return `${hours}h ${mins}m`;
  };

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map((status) => (
        <div
          key={status}
          className="kanban-column"
          onDragOver={handleDragOver}
          onDrop={(e) => handleDrop(e, status)}
        >
          <h3 className="column-title">
            {status === 'todo' ? 'To Do' : status === 'in-progress' ? 'In Progress' : 'Done'}
            <span className="task-count">({tasks[status].length})</span>
          </h3>
          
          <div className="tasks-container">
            {tasks[status].map((task) => (
              <div
                key={task._id}
                className={`task-card priority-${task.priority}`}
                draggable
                onDragStart={(e) => handleDragStart(e, task._id, status)}
              >
                <h4>{task.title}</h4>
                {task.description && <p className="task-description">{task.description}</p>}
                
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
                
                {task.timeSpent > 0 && (
                  <div className="time-spent">
                    ⏱️ {formatTime(task.timeSpent)}
                  </div>
                )}
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

### Time Tracker Component

```javascript
