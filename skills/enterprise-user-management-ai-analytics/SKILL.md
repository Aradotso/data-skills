---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket routing, risk detection, and burnout analysis
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics for user management
  - implement task tracking with burnout detection
  - create admin dashboard with user analytics
  - set up ticket classification system
  - configure JWT authentication for user management
  - build kanban board with time tracking
  - deploy user management system with ML service
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user/task management with AI-powered insights. It provides role-based access control, task tracking with Kanban boards, support ticket management, and ML-driven features including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

**Key Components:**
- **Frontend**: React.js dashboard for users and admins
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based AI analytics engine
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.4
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
REACT_APP_ML_API_URL=http://localhost:8000
EOF

npm start
```

## Architecture

### Backend API Structure

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Database connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
});

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
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  avatar: String,
  isActive: { type: Boolean, default: true },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now }
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

## Authentication

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization && 
      req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ 
      success: false, 
      message: 'Not authorized to access this route' 
    });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (err) {
    return res.status(401).json({ 
      success: false, 
      message: 'Token is invalid or expired' 
    });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        message: `User role ${req.user.role} is not authorized`
      });
    }
    next();
  };
};
```

### Auth Routes

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const userExists = await User.findOne({ $or: [{ email }, { username }] });
    if (userExists) {
      return res.status(400).json({ 
        success: false, 
        message: 'User already exists' 
      });
    }
    
    const user = await User.create({ username, email, password, role });
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });
    
    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user || !(await user.comparePassword(password))) {
      return res.status(401).json({ 
        success: false, 
        message: 'Invalid credentials' 
      });
    }
    
    user.lastLogin = new Date();
    await user.save();
    
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });
    
    res.json({
      success: true,
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

## Task Management

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  status: { 
    type: String, 
    enum: ['todo', 'in-progress', 'done'], 
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
  timeSpent: { type: Number, default: 0 }, // in minutes
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Task Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { protect, authorize } = require('../middleware/auth');

// Get all tasks (with filtering)
router.get('/', protect, async (req, res) => {
  try {
    const { status, assignedTo, priority } = req.query;
    const filter = {};
    
    if (status) filter.status = status;
    if (priority) filter.priority = priority;
    if (assignedTo) filter.assignedTo = assignedTo;
    
    // Non-admin users only see their tasks
    if (req.user.role !== 'admin') {
      filter.assignedTo = req.user._id;
    }
    
    const tasks = await Task.find(filter)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username')
      .sort('-createdAt');
      
    res.json({ success: true, count: tasks.length, data: tasks });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Create task (admin only)
router.post('/', protect, authorize('admin'), async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user._id
    });
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Update task
router.put('/:id', protect, async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ 
        success: false, 
        message: 'Task not found' 
      });
    }
    
    // Check authorization
    if (req.user.role !== 'admin' && 
        task.assignedTo.toString() !== req.user._id.toString()) {
      return res.status(403).json({ 
        success: false, 
        message: 'Not authorized' 
      });
    }
    
    const updatedTask = await Task.findByIdAndUpdate(
      req.params.id,
      { ...req.body, updatedAt: Date.now() },
      { new: true, runValidators: true }
    );
    
    res.json({ success: true, data: updatedTask });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Update time spent
router.patch('/:id/time', protect, async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeSpent: timeSpent } },
      { new: true }
    );
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

## Support Tickets

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { 
    type: String, 
    enum: ['bug', 'feature', 'support', 'other'], 
    required: true 
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
    suggestedPriority: String
  },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### Ticket Routes with AI Integration

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { protect, authorize } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, category } = req.body;
    
    // Call ML service for AI classification
    let aiClassification = {};
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/classify-ticket`,
        { title, description, category }
      );
      aiClassification = mlResponse.data;
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }
    
    const ticket = await Ticket.create({
      title,
      description,
      category,
      createdBy: req.user._id,
      aiClassification,
      priority: aiClassification.suggestedPriority || 'medium'
    });
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Get tickets
router.get('/', protect, async (req, res) => {
  try {
    const filter = {};
    
    // Non-admin users only see their tickets
    if (req.user.role !== 'admin') {
      filter.createdBy = req.user._id;
    }
    
    const tickets = await Ticket.find(filter)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort('-createdAt');
      
    res.json({ success: true, count: tickets.length, data: tickets });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

## ML Service (AI Analytics)

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, List
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="User Management AI Analytics")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    category: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_frequency: int
    failed_logins: int
    unusual_hours: int
    data_access_volume: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_count: int
    overdue_tasks: int
    avg_task_completion_time: float
    overtime_hours: float
    days_without_break: int

@app.get("/")
async def root():
    return {"message": "User Management AI Analytics API", "status": "running"}

@app.post("/api/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """AI-based ticket classification and priority assignment"""
    try:
        # Simple rule-based classification (replace with trained model)
        keywords_high = ['critical', 'urgent', 'crash', 'down', 'broken']
        keywords_medium = ['issue', 'problem', 'error', 'bug']
        
        text = (request.title + " " + request.description).lower()
        
        confidence = 0.85
        suggested_priority = 'low'
        
        if any(keyword in text for keyword in keywords_high):
            suggested_priority = 'high'
            confidence = 0.92
        elif any(keyword in text for keyword in keywords_medium):
            suggested_priority = 'medium'
            confidence = 0.87
        
        return {
            "category": request.category,
            "confidence": confidence,
            "suggestedPriority": suggested_priority
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict security risk based on user behavior"""
    try:
        # Calculate risk score (0-100)
        risk_score = 0
        risk_factors = []
        
        if request.failed_logins > 3:
            risk_score += 30
            risk_factors.append("Multiple failed login attempts")
        
        if request.unusual_hours > 5:
            risk_score += 25
            risk_factors.append("Unusual activity hours")
        
        if request.data_access_volume > 1000:
            risk_score += 20
            risk_factors.append("High data access volume")
        
        if request.login_frequency > 50:
            risk_score += 15
            risk_factors.append("Abnormal login frequency")
        
        risk_level = "low"
        if risk_score > 60:
            risk_level = "high"
        elif risk_score > 30:
            risk_level = "medium"
        
        return {
            "user_id": request.user_id,
            "risk_score": min(risk_score, 100),
            "risk_level": risk_level,
            "risk_factors": risk_factors,
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Detect employee burnout based on workload patterns"""
    try:
        burnout_score = 0
        indicators = []
        
        # Task overload
        if request.tasks_count > 20:
            burnout_score += 25
            indicators.append("High task count")
        
        # Overdue tasks
        if request.overdue_tasks > 5:
            burnout_score += 30
            indicators.append("Multiple overdue tasks")
        
        # Long completion times
        if request.avg_task_completion_time > 8:
            burnout_score += 20
            indicators.append("Extended task completion times")
        
        # Overtime
        if request.overtime_hours > 10:
            burnout_score += 15
            indicators.append("Excessive overtime")
        
        # No breaks
        if request.days_without_break > 14:
            burnout_score += 10
            indicators.append("No recent breaks")
        
        risk_level = "low"
        if burnout_score > 60:
            risk_level = "high"
        elif burnout_score > 35:
            risk_level = "medium"
        
        recommendations = []
        if risk_level in ["medium", "high"]:
            recommendations = [
                "Consider redistributing workload",
                "Schedule time off or breaks",
                "Review task priorities",
                "Provide additional support or resources"
            ]
        
        return {
            "user_id": request.user_id,
            "burnout_score": min(burnout_score, 100),
            "risk_level": risk_level,
            "indicators": indicators,
            "recommendations": recommendations,
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/predict-project-delay")
async def predict_project_delay(
    completed_tasks: int,
    total_tasks: int,
    days_elapsed: int,
    estimated_days: int,
    team_size: int
):
    """Predict likelihood of project delay"""
    try:
        completion_rate = completed_tasks / total_tasks if total_tasks > 0 else 0
        time_progress = days_elapsed / estimated_days if estimated_days > 0 else 0
        
        # Calculate delay probability
        if completion_rate < time_progress - 0.15:
            delay_probability = 0.85
            delay_estimate_days = int((estimated_days - days_elapsed) * 0.3)
        elif completion_rate < time_progress:
            delay_probability = 0.60
            delay_estimate_days = int((estimated_days - days_elapsed) * 0.15)
        else:
            delay_probability = 0.25
            delay_estimate_days = 0
        
        return {
            "delay_probability": delay_probability,
            "estimated_delay_days": delay_estimate_days,
            "completion_rate": round(completion_rate, 2),
            "time_progress": round(time_progress, 2),
            "status": "at_risk" if delay_probability > 0.5 else "on_track"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### ML Service Requirements

```txt
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
numpy==1.24.3
scikit-learn==1.3.2
river==0.19.0
joblib==1.3.2
python-dotenv==1.0.0
pymongo==4.6.0
```

## Frontend Integration

### API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

// Create axios instance with auth
const api = axios.create({
  baseURL: API_URL
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auth API
export const authAPI = {
  login: (credentials) => api.post('/api/auth/login', credentials),
  register: (userData) => api.post('/api/auth/register', userData),
  getCurrentUser: () => api.get('/api/auth/me')
};

// User API
export const userAPI = {
  getAll: () => api.get('/api/users'),
  getById: (id) => api.get(`/api/users/${id}`),
  create: (userData) => api.post('/api/users', userData),
  update: (id, userData) => api.put(`/api/users/${id}`, userData),
  delete: (id) => api.delete(`/api/users/${id}`)
};

// Task API
export const taskAPI = {
  getAll: (filters) => api.get('/api/tasks', { params: filters }),
  getById: (id) => api.get(`/api/tasks/${id}`),
  create: (taskData) => api.post('/api/tasks', taskData),
  update: (id, taskData) => api.put(`/api/tasks/${id}`, taskData),
  updateTime: (id, timeSpent) => 
    api.patch(`/api/tasks/${id}/time`, { timeSpent }),
  delete: (id) => api.delete(`/api/tasks/${id}`)
};

// Ticket API
export const ticketAPI = {
  getAll: () => api.get('/api/tickets'),
  create: (ticketData) => api.post('/api/tickets', ticketData),
  update: (id, ticketData) => api.put(`/api/tickets/${id}`, ticketData)
};

// ML API
export const mlAPI = {
  classifyTicket: (ticketData) => 
    axios.post(`${ML_API_URL}/api/classify-ticket`, ticketData),
  predictRisk: (userData) => 
    axios.post(`${ML_API_URL}/api/predict-risk`, userData),
  analyzeBurnout: (workloadData) => 
    axios.post(`${ML_API_URL}/api/analyze-burnout`, workloadData),
  predictProjectDelay: (projectData) => 
    axios.post(`${ML_API_URL}/api/predict-project-delay`, projectData)
};
```

### React Task Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskAPI.getAll();
      const grouped = response.data.data.reduce((acc, task) => {
        if (!acc[task.status]) acc[task.status] = [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], 'in-progress': [], done: [] });
      
      setTasks(grouped);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskAPI.update(taskId, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority priority-${task.priority}`}>
          {task.priority}
        </span>
        <span>Time: {Math.floor(task.timeSpent / 60)}h {task.timeSpent % 60}m</span>
      </div>
      <select 
        value={task.status} 
        onChange={(e) => handleStatusChange(task._id, e.target.value)}
      >
        <option value="todo">To Do</option>
        <option value="in-progress">In Progress</option>
        <option value="done">Done</option>
      </select>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {Object.entries(tasks).map(([status, taskList]) => (
        <div key={status} className="kanban-column">
          <h3>{status.replace('-', ' ').toUpperCase()}</h3>
          <div className="task-list">
            {taskList.map(task => (
              <TaskCard key={task._id} task={task} />
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default TaskBoard;
```

### Burnout Analysis Component

```javascript
// frontend/src/components/BurnoutAnalysis.jsx
import React, { useState, useEffect } from 'react';
import { mlAPI, taskAPI } from '../services/api';

const BurnoutAnalysis = ({ userId }) => {
  const [analysis, setAnalysis] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    analyzeBurnout();
  }, [userId]);

  const analyzeBurnout = async () => {
    try {
      // Fetch user task data
      const tasksResponse = await taskAPI.getAll({ assignedTo: userId });
      const tasks = tasksResponse.data.data;
      
      const overdueTasks = tasks.filter(t => 
        new Date(t.dueDate) < new Date() && t.status !== 'done'
      ).length;
      
      const avgCompletionTime = tasks
        .filter(t => t.status === 'done')
        .reduce((acc, t) => acc + t.timeSpent, 0) / 
        (tasks.filter(t => t.status === 'done').length || 1) / 60;
      
      // Call ML service
      const mlResponse = await mlAPI.analyzeBurnout({
        user_id: userId,
        tasks_count: tasks.length,
        overdue_tasks: overdueTasks,
        avg_task_completion_time: avgCompletionTime,
        overtime_hours: 12, // Calculate from time tracking
        days_without_break: 10
      });
      
      setAnalysis(mlResponse.data);
      setLoading(false);
    } catch (error) {
      console.error('Error analyzing burnout:', error);
      setLoading(false);
    }
  };

  if (loading) return 
