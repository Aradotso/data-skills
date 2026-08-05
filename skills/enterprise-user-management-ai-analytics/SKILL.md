---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and task optimization
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "add AI-powered ticket classification"
  - "build admin panel with user management"
  - "integrate burnout detection and risk analysis"
  - "deploy user management system with AI features"
  - "configure kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What It Does

Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards, time tracking, assignment workflows
- **Support Tickets**: AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Separate admin and user interfaces with real-time insights

The system uses React.js frontend, Node.js/Express backend, MongoDB for data persistence, and FastAPI with scikit-learn/River for ML services.

## Installation

### Complete System Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env  # Configure your environment variables
npm start  # Runs on http://localhost:5000

# ML Service setup (separate terminal)
cd ml-service
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup (separate terminal)
cd frontend
npm install
npm start  # Runs on http://localhost:3000
```

### Environment Configuration

**Backend (.env)**:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
CORS_ORIGINS=http://localhost:3000,http://localhost:5000
```

**Frontend (.env)**:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

## Key Architecture

### Backend API Structure

The Node.js backend typically follows this structure:

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const jwt = require('jsonwebtoken');
require('dotenv').config();

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Database connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
})
.then(() => console.log('MongoDB connected'))
.catch(err => console.error('MongoDB connection error:', err));

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/tasks', require('./routes/tasks'));
app.use('/api/tickets', require('./routes/tickets'));
app.use('/api/analytics', require('./routes/analytics'));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  department: { type: String },
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  tasksAssigned: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Task' }],
  workloadScore: { type: Number, default: 0 },
  riskScore: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  lastLogin: { type: Date }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');
    
    if (!user) {
      return res.status(401).json({ message: 'User not found' });
    }

    req.user = user;
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

module.exports = { auth, adminOnly };
```

### Task Model and Routes

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  status: { type: String, enum: ['todo', 'inProgress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: { type: Date },
  timeSpent: { type: Number, default: 0 }, // in minutes
  estimatedTime: { type: Number }, // in minutes
  tags: [{ type: String }],
  createdAt: { type: Date, default: Date.now },
  completedAt: { type: Date }
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { auth, adminOnly } = require('../middleware/auth');

// Get all tasks (admin) or user's tasks
router.get('/', auth, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user._id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create new task
router.post('/', auth, adminOnly, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id
    });
    
    await task.save();
    await task.populate('assignedTo', 'name email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
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
    
    // Check permissions
    if (req.user.role !== 'admin' && task.assignedTo.toString() !== req.user._id.toString()) {
      return res.status(403).json({ message: 'Access denied' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Update time spent
router.patch('/:id/time', auth, async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeSpent: timeSpent } },
      { new: true }
    );
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI ML Service Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
import joblib
import os

app = FastAPI(title="Enterprise UMS AI Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=os.getenv("CORS_ORIGINS", "http://localhost:3000").split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models storage
models = {}

class UserBehavior(BaseModel):
    user_id: str
    login_frequency: float
    task_completion_rate: float
    avg_time_per_task: float
    failed_login_attempts: int
    workload_score: float
    overtime_hours: float

class TicketData(BaseModel):
    title: str
    description: str
    priority: Optional[str] = "medium"

class ProjectData(BaseModel):
    tasks_total: int
    tasks_completed: int
    avg_completion_time: float
    team_size: int
    days_remaining: int

# Risk Detection
@app.post("/api/ml/risk-detection")
async def detect_risk(user: UserBehavior):
    """Detect user risk based on behavior patterns"""
    try:
        # Feature engineering
        features = np.array([[
            user.login_frequency,
            user.task_completion_rate,
            user.avg_time_per_task,
            user.failed_login_attempts,
            user.workload_score
        ]])
        
        # Simple rule-based risk scoring
        risk_score = 0
        
        if user.failed_login_attempts > 3:
            risk_score += 30
        if user.task_completion_rate < 0.5:
            risk_score += 25
        if user.login_frequency < 0.3:
            risk_score += 20
        if user.workload_score > 80:
            risk_score += 15
        if user.avg_time_per_task > 120:
            risk_score += 10
            
        risk_level = "low"
        if risk_score > 50:
            risk_level = "high"
        elif risk_score > 25:
            risk_level = "medium"
        
        return {
            "user_id": user.user_id,
            "risk_score": min(risk_score, 100),
            "risk_level": risk_level,
            "factors": {
                "failed_logins": user.failed_login_attempts > 3,
                "low_completion_rate": user.task_completion_rate < 0.5,
                "irregular_login": user.login_frequency < 0.3,
                "overloaded": user.workload_score > 80
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Anomaly Detection
@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(behaviors: List[UserBehavior]):
    """Detect anomalies in user behavior patterns"""
    try:
        if len(behaviors) < 2:
            return {"anomalies": []}
        
        # Prepare features
        features = np.array([[
            b.login_frequency,
            b.task_completion_rate,
            b.avg_time_per_task,
            b.failed_login_attempts,
            b.workload_score
        ] for b in behaviors])
        
        # Use Isolation Forest for anomaly detection
        clf = IsolationForest(contamination=0.1, random_state=42)
        predictions = clf.fit_predict(features)
        
        anomalies = []
        for idx, (pred, behavior) in enumerate(zip(predictions, behaviors)):
            if pred == -1:  # Anomaly detected
                anomalies.append({
                    "user_id": behavior.user_id,
                    "anomaly_score": float(clf.score_samples([features[idx]])[0]),
                    "behavior": behavior.dict()
                })
        
        return {"anomalies": anomalies, "total_checked": len(behaviors)}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout Detection
@app.post("/api/ml/burnout-detection")
async def detect_burnout(user: UserBehavior):
    """Analyze user workload for burnout risk"""
    try:
        burnout_score = 0
        factors = []
        
        # Workload analysis
        if user.workload_score > 80:
            burnout_score += 30
            factors.append("High workload score")
        
        # Overtime analysis
        if user.overtime_hours > 10:
            burnout_score += 25
            factors.append("Excessive overtime")
        
        # Task completion rate (too high might indicate overwork)
        if user.task_completion_rate > 0.95:
            burnout_score += 15
            factors.append("Unsustainable completion rate")
        
        # Average time per task
        if user.avg_time_per_task > 150:
            burnout_score += 20
            factors.append("Extended task duration")
        
        # Login frequency (too high might indicate stress)
        if user.login_frequency > 0.9:
            burnout_score += 10
            factors.append("Very frequent logins")
        
        burnout_level = "low"
        if burnout_score > 60:
            burnout_level = "high"
        elif burnout_score > 30:
            burnout_level = "medium"
        
        recommendations = []
        if burnout_level == "high":
            recommendations = [
                "Redistribute workload immediately",
                "Schedule rest period",
                "Review task priorities"
            ]
        elif burnout_level == "medium":
            recommendations = [
                "Monitor workload closely",
                "Consider task delegation"
            ]
        
        return {
            "user_id": user.user_id,
            "burnout_score": min(burnout_score, 100),
            "burnout_level": burnout_level,
            "factors": factors,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Ticket Classification
@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketData):
    """Classify support ticket and suggest routing"""
    try:
        # Simple keyword-based classification
        text = (ticket.title + " " + ticket.description).lower()
        
        # Department routing
        if any(word in text for word in ['password', 'login', 'access', 'account']):
            department = "IT Support"
            priority = "high"
        elif any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            department = "Development"
            priority = "high"
        elif any(word in text for word in ['feature', 'enhancement', 'request']):
            department = "Product"
            priority = "medium"
        elif any(word in text for word in ['billing', 'payment', 'invoice']):
            department = "Finance"
            priority = "medium"
        else:
            department = "General Support"
            priority = ticket.priority
        
        # Urgency detection
        urgent_keywords = ['urgent', 'critical', 'asap', 'immediately', 'emergency']
        if any(word in text for word in urgent_keywords):
            priority = "high"
        
        return {
            "department": department,
            "priority": priority,
            "category": _categorize_ticket(text),
            "estimated_resolution_time": _estimate_time(department, priority)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def _categorize_ticket(text: str) -> str:
    """Helper to categorize ticket"""
    if 'bug' in text or 'error' in text:
        return "technical"
    elif 'feature' in text or 'request' in text:
        return "feature_request"
    elif 'question' in text or 'how to' in text:
        return "inquiry"
    else:
        return "general"

def _estimate_time(department: str, priority: str) -> str:
    """Helper to estimate resolution time"""
    time_map = {
        ("IT Support", "high"): "2-4 hours",
        ("IT Support", "medium"): "4-8 hours",
        ("Development", "high"): "1-2 days",
        ("Development", "medium"): "2-5 days",
        ("Product", "medium"): "3-7 days",
        ("Finance", "medium"): "1-2 days"
    }
    return time_map.get((department, priority), "1-3 days")

# Project Delay Prediction
@app.post("/api/ml/predict-delay")
async def predict_delay(project: ProjectData):
    """Predict if project will be delayed"""
    try:
        completion_rate = project.tasks_completed / project.tasks_total if project.tasks_total > 0 else 0
        tasks_remaining = project.tasks_total - project.tasks_completed
        
        # Simple prediction logic
        required_rate = tasks_remaining / project.days_remaining if project.days_remaining > 0 else float('inf')
        current_rate = project.tasks_completed / (project.avg_completion_time or 1)
        
        delay_probability = 0
        if required_rate > current_rate:
            delay_probability = min((required_rate / current_rate) * 50, 90)
        else:
            delay_probability = max(20 - (current_rate - required_rate) * 10, 5)
        
        will_delay = delay_probability > 50
        
        return {
            "will_delay": will_delay,
            "delay_probability": round(delay_probability, 2),
            "completion_rate": round(completion_rate * 100, 2),
            "tasks_remaining": tasks_remaining,
            "recommended_daily_rate": round(required_rate, 2),
            "suggestions": _get_project_suggestions(will_delay, project)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def _get_project_suggestions(will_delay: bool, project: ProjectData) -> List[str]:
    """Helper to generate project suggestions"""
    suggestions = []
    if will_delay:
        suggestions.append("Increase team size or reallocate resources")
        suggestions.append("Review and prioritize critical tasks")
        if project.avg_completion_time > 3:
            suggestions.append("Identify bottlenecks in task completion")
    else:
        suggestions.append("Current pace is on track")
        suggestions.append("Monitor progress regularly")
    return suggestions

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "Enterprise UMS ML Service"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Frontend Integration

### React API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_URL = process.env.REACT_APP_ML_URL;

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
  (error) => Promise.reject(error)
);

// Auth services
export const authService = {
  login: async (email, password) => {
    const response = await api.post('/auth/login', { email, password });
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    return response.data;
  },
  
  register: async (userData) => {
    const response = await api.post('/auth/register', userData);
    return response.data;
  },
  
  logout: () => {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  },
  
  getCurrentUser: () => {
    return JSON.parse(localStorage.getItem('user'));
  }
};

// Task services
export const taskService = {
  getAllTasks: async () => {
    const response = await api.get('/tasks');
    return response.data;
  },
  
  createTask: async (taskData) => {
    const response = await api.post('/tasks', taskData);
    return response.data;
  },
  
  updateTaskStatus: async (taskId, status) => {
    const response = await api.patch(`/tasks/${taskId}/status`, { status });
    return response.data;
  },
  
  updateTaskTime: async (taskId, timeSpent) => {
    const response = await api.patch(`/tasks/${taskId}/time`, { timeSpent });
    return response.data;
  }
};

// ML services
export const mlService = {
  detectRisk: async (userData) => {
    const response = await axios.post(`${ML_URL}/api/ml/risk-detection`, userData);
    return response.data;
  },
  
  detectBurnout: async (userData) => {
    const response = await axios.post(`${ML_URL}/api/ml/burnout-detection`, userData);
    return response.data;
  },
  
  classifyTicket: async (ticketData) => {
    const response = await axios.post(`${ML_URL}/api/ml/classify-ticket`, ticketData);
    return response.data;
  },
  
  predictDelay: async (projectData) => {
    const response = await axios.post(`${ML_URL}/api/ml/predict-delay`, projectData);
    return response.data;
  }
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
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const data = await taskService.getAllTasks();
      
      // Group tasks by status
      const grouped = {
        todo: data.filter(t => t.status === 'todo'),
        inProgress: data.filter(t => t.status === 'inProgress'),
        done: data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const handleDragStart = (e, task) => {
    e.dataTransfer.setData('taskId', task._id);
    e.dataTransfer.setData('currentStatus', task.status);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');
    
    if (currentStatus !== newStatus) {
      try {
        await taskService.updateTaskStatus(taskId, newStatus);
        await fetchTasks(); // Refresh tasks
      } catch (error) {
        console.error('Error updating task:', error);
      }
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const renderColumn = (status, title, taskList) => (
    <div 
      className="kanban-column"
      onDrop={(e) => handleDrop(e, status)}
      onDragOver={handleDragOver}
    >
      <div className="column-header">
        <h3>{title}</h3>
        <span className="task-count">{taskList.length}</span>
      </div>
      
      <div className="task-list">
        {taskList.map(task => (
          <div
            key={task._id}
            className={`task-card priority-${task.priority}`}
            draggable
            onDragStart={(e) => handleDragStart(e, task)}
          >
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            
            <div className="task-meta">
              <span className="priority-badge">{task.priority}</span>
              {task.dueDate && (
                <span className="due-date">
                  Due: {new Date(task.dueDate).toLocaleDateString()}
                </span>
              )}
            </div>
            
            {task.timeSpent > 0 && (
              <div className="time-spent">
                ⏱️ {Math.floor(task.timeSpent / 60)}h {task.timeSpent % 60}m
              </div>
            )}
            
            {task.assignedTo && (
              <div className="assigned-to">
                👤 {task.assignedTo.name}
              </div>
            )}
          </div>
        ))}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do', tasks.todo)}
      {renderColumn('inProgress', 'In Progress', tasks.inProgress)}
      {renderColumn('done', 'Done', tasks.done)}
    </div>
  );
};

export default KanbanBoard;
```

### User Dashboard with AI Analytics

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import { taskService, mlService } from '../services/api';
import { authService } from '../services/api';

const UserDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [tasks, setTasks] = useState([]);
  const [burnoutData, setBurnoutData] = useState(null);
  const user = authService.getCurrentUser();

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const tasksData = await taskService.getAllTasks();
      setTasks(tasksData);

      // Calculate user behavior metrics
      const totalTasks = tasksData.length;
      const completedTasks = tasksData.filter(t => t.status === 'done').length;
      const avgTime = tasksData.reduce((acc, t) => acc + (t.timeSpent || 0), 0) / totalTasks || 0;

      // Get AI analytics
      const userBehavior = {
        user_id: user.id,
        login_frequency: 0.75, // This would come from backend tracking
        task_completion_rate: completedTasks / totalTasks || 0,
        avg_time_per_task: avgTime,
        failed_login_attempts: 0,
        workload_score: totalTasks * 5, // Simplified calculation
        overtime_hours: 5 // This would come from time tracking
      };

      const burnout = await mlService.detectBurnout(userBehavior);
      setBurnoutData(burnout);

      setAnalytics({
        totalTasks,
        completedTasks,
        inProgress: tasksData.filter(t => t.status === 'inProgress').length,
        avgTime: Math.round(avgTime)
      });
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
    }
  };

  return (
    <div className="user-dashboard">
      <h1>Welcome, {user?.name}</h1>

      {/* Task Statistics */}
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Tasks</h3>
          <p className="stat-number">{analytics?.totalTasks || 0}</p>
        </div>
        <div className="stat-card">
          <h3>Completed</h3>
          <p className="stat-number">{analytics?.completedTasks || 0}</p>
        </div>
        <div className="stat-card">
          <h3>In Progress
