---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, task management, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement role-based user administration"
  - "create task tracking with AI insights"
  - "build support ticket system with ML"
  - "add anomaly detection to user system"
  - "deploy user management with FastAPI ML"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise application that combines user management, task tracking, and support ticketing with AI-powered analytics including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## What This Project Does

This system provides:
- **User Management**: Role-based access control (Admin/User), authentication via JWT
- **Task Management**: Kanban board with time tracking and progress monitoring
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk detection, anomaly detection, burnout prediction, project delay forecasting
- **Dashboards**: Admin oversight and user performance tracking

**Stack**: React frontend, Node.js/Express backend, FastAPI ML service, MongoDB database

## Installation

### Prerequisites

```bash
# Ensure you have installed:
node --version  # v14+
python --version  # 3.8+
mongod --version  # MongoDB 4.4+
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
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=\${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
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

# Start frontend
npm start
# Runs at http://localhost:3000
```

## Key Architecture

### Backend API Structure

The Node.js backend provides RESTful APIs:

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

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/tasks', require('./routes/tasks'));
app.use('/api/tickets', require('./routes/tickets'));
app.use('/api/analytics', require('./routes/analytics'));

// MongoDB Connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => console.log('MongoDB connected'))
  .catch(err => console.error('MongoDB error:', err));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ message: 'No token, authorization denied' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Token is not valid' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
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
  department: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  tasksCompleted: { type: Number, default: 0 },
  workloadScore: { type: Number, default: 0 },
  riskScore: { type: Number, default: 0 },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Auth Routes

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const router = express.Router();

// Register
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const user = new User({ name, email, password, role });
    await user.save();
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ token, user: { id: user._id, name, email, role } });
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
    
    const isMatch = await user.comparePassword(password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    user.lastLogin = new Date();
    await user.save();
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ token, user: { id: user._id, name: user.name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Task Management Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

const router = express.Router();

// Get user tasks
router.get('/my-tasks', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('assignedBy', 'name email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task (Admin only)
router.post('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.userId,
      priority,
      dueDate,
      status: 'todo'
    });
    
    await task.save();
    await task.populate('assignedTo assignedBy', 'name email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
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
    res.status(500).json({ message: error.message });
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
import numpy as np
from typing import List, Optional
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

# Load models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class UserBehavior(BaseModel):
    user_id: str
    login_count: int
    failed_login_attempts: int
    avg_session_duration: float
    tasks_completed: int
    tasks_overdue: int
    workload_hours: float
    last_activity_days: int

class TicketData(BaseModel):
    title: str
    description: str
    priority: Optional[str] = "medium"

@app.get("/")
def root():
    return {"service": "Enterprise UMS ML Service", "status": "running"}

@app.post("/api/ml/risk-prediction")
async def predict_risk(data: UserBehavior):
    """Predict user risk score based on behavior patterns"""
    try:
        # Feature engineering
        features = np.array([[
            data.failed_login_attempts,
            data.login_count,
            data.avg_session_duration,
            data.tasks_overdue / max(data.tasks_completed, 1),
            data.last_activity_days,
            data.workload_hours
        ]])
        
        # Simple risk scoring algorithm
        risk_score = 0
        
        # Failed logins indicate security risk
        if data.failed_login_attempts > 5:
            risk_score += 30
        elif data.failed_login_attempts > 2:
            risk_score += 15
            
        # Overdue tasks indicate performance risk
        overdue_ratio = data.tasks_overdue / max(data.tasks_completed, 1)
        risk_score += min(overdue_ratio * 40, 40)
        
        # Inactivity risk
        if data.last_activity_days > 7:
            risk_score += 20
        elif data.last_activity_days > 3:
            risk_score += 10
            
        # Normalize to 0-100
        risk_score = min(risk_score, 100)
        
        risk_level = "low" if risk_score < 30 else "medium" if risk_score < 70 else "high"
        
        return {
            "risk_score": round(risk_score, 2),
            "risk_level": risk_level,
            "factors": {
                "security": data.failed_login_attempts > 2,
                "performance": overdue_ratio > 0.3,
                "engagement": data.last_activity_days > 7
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(data: UserBehavior):
    """Detect anomalous user behavior"""
    try:
        anomalies = []
        
        # Check for unusual login patterns
        if data.failed_login_attempts > 10:
            anomalies.append({
                "type": "security",
                "description": "Excessive failed login attempts",
                "severity": "high"
            })
        
        # Check for unusual workload
        if data.workload_hours > 60:
            anomalies.append({
                "type": "workload",
                "description": "Excessive work hours detected",
                "severity": "medium"
            })
        
        # Check for inactivity
        if data.last_activity_days > 14 and data.tasks_overdue > 0:
            anomalies.append({
                "type": "engagement",
                "description": "Prolonged inactivity with pending tasks",
                "severity": "medium"
            })
        
        is_anomalous = len(anomalies) > 0
        
        return {
            "is_anomalous": is_anomalous,
            "anomalies": anomalies,
            "confidence": 0.85 if is_anomalous else 0.95
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(data: UserBehavior):
    """Detect employee burnout risk"""
    try:
        burnout_score = 0
        
        # High workload indicator
        if data.workload_hours > 50:
            burnout_score += 40
        elif data.workload_hours > 40:
            burnout_score += 20
        
        # Task completion rate
        completion_rate = data.tasks_completed / max(data.tasks_completed + data.tasks_overdue, 1)
        if completion_rate < 0.7:
            burnout_score += 30
        
        # Session duration (very short or very long sessions indicate stress)
        if data.avg_session_duration < 1 or data.avg_session_duration > 10:
            burnout_score += 15
        
        burnout_score = min(burnout_score, 100)
        burnout_level = "low" if burnout_score < 40 else "moderate" if burnout_score < 70 else "high"
        
        return {
            "burnout_score": round(burnout_score, 2),
            "burnout_level": burnout_level,
            "recommendations": [
                "Consider workload redistribution" if data.workload_hours > 50 else None,
                "Schedule wellness check-in" if burnout_score > 70 else None,
                "Review task priorities" if data.tasks_overdue > 5 else None
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/ticket-classification")
async def classify_ticket(data: TicketData):
    """Classify support ticket and route to appropriate department"""
    try:
        text = (data.title + " " + data.description).lower()
        
        # Simple keyword-based classification
        if any(word in text for word in ['password', 'login', 'access', 'authentication']):
            category = "IT Support"
            department = "IT"
        elif any(word in text for word in ['payroll', 'salary', 'leave', 'benefit']):
            category = "HR"
            department = "Human Resources"
        elif any(word in text for word in ['bug', 'error', 'crash', 'feature']):
            category = "Development"
            department = "Engineering"
        elif any(word in text for word in ['network', 'connection', 'vpn', 'wifi']):
            category = "Infrastructure"
            department = "IT"
        else:
            category = "General"
            department = "Support"
        
        # Determine urgency
        urgent_keywords = ['urgent', 'critical', 'emergency', 'down', 'broken']
        is_urgent = any(word in text for word in urgent_keywords) or data.priority == "high"
        
        return {
            "category": category,
            "department": department,
            "priority": "high" if is_urgent else data.priority,
            "estimated_resolution_hours": 2 if is_urgent else 24,
            "confidence": 0.82
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Frontend Integration

### API Client Setup

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

// Create axios instance with auth interceptor
const api = axios.create({
  baseURL: API_URL,
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auth services
export const authService = {
  login: (credentials) => api.post('/api/auth/login', credentials),
  register: (userData) => api.post('/api/auth/register', userData),
  logout: () => localStorage.removeItem('token'),
};

// User services
export const userService = {
  getUsers: () => api.get('/api/users'),
  getUserById: (id) => api.get(`/api/users/${id}`),
  createUser: (userData) => api.post('/api/users', userData),
  updateUser: (id, userData) => api.patch(`/api/users/${id}`, userData),
  deleteUser: (id) => api.delete(`/api/users/${id}`),
};

// Task services
export const taskService = {
  getMyTasks: () => api.get('/api/tasks/my-tasks'),
  getAllTasks: () => api.get('/api/tasks'),
  createTask: (taskData) => api.post('/api/tasks', taskData),
  updateTaskStatus: (id, status) => api.patch(`/api/tasks/${id}/status`, { status }),
};

// ML services
export const mlService = {
  predictRisk: (userBehavior) => axios.post(`${ML_API_URL}/api/ml/risk-prediction`, userBehavior),
  detectAnomaly: (userBehavior) => axios.post(`${ML_API_URL}/api/ml/anomaly-detection`, userBehavior),
  detectBurnout: (userBehavior) => axios.post(`${ML_API_URL}/api/ml/burnout-detection`, userBehavior),
  classifyTicket: (ticketData) => axios.post(`${ML_API_URL}/api/ml/ticket-classification`, ticketData),
};
```

### User Dashboard Component

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import { taskService, mlService } from '../services/api';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      const { data: allTasks } = await taskService.getMyTasks();
      
      // Group tasks by status
      const groupedTasks = {
        todo: allTasks.filter(t => t.status === 'todo'),
        inProgress: allTasks.filter(t => t.status === 'in_progress'),
        done: allTasks.filter(t => t.status === 'done')
      };
      
      setTasks(groupedTasks);
      
      // Get AI analytics
      const userBehavior = {
        user_id: localStorage.getItem('userId'),
        login_count: 45,
        failed_login_attempts: 0,
        avg_session_duration: 3.5,
        tasks_completed: groupedTasks.done.length,
        tasks_overdue: allTasks.filter(t => new Date(t.dueDate) < new Date()).length,
        workload_hours: 38,
        last_activity_days: 0
      };
      
      const [riskData, burnoutData] = await Promise.all([
        mlService.predictRisk(userBehavior),
        mlService.detectBurnout(userBehavior)
      ]);
      
      setAnalytics({
        risk: riskData.data,
        burnout: burnoutData.data
      });
      
      setLoading(false);
    } catch (error) {
      console.error('Error loading dashboard:', error);
      setLoading(false);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await taskService.updateTaskStatus(taskId, newStatus);
      loadDashboardData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  if (loading) return <div>Loading dashboard...</div>;

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {/* Analytics Overview */}
      {analytics && (
        <div className="analytics-cards">
          <div className={`card risk-${analytics.risk.risk_level}`}>
            <h3>Risk Score</h3>
            <p className="score">{analytics.risk.risk_score}</p>
            <span className="level">{analytics.risk.risk_level}</span>
          </div>
          
          <div className={`card burnout-${analytics.burnout.burnout_level}`}>
            <h3>Burnout Level</h3>
            <p className="score">{analytics.burnout.burnout_score}</p>
            <span className="level">{analytics.burnout.burnout_level}</span>
          </div>
        </div>
      )}
      
      {/* Kanban Board */}
      <div className="kanban-board">
        <div className="column">
          <h2>To Do ({tasks.todo.length})</h2>
          {tasks.todo.map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onMove={(id) => moveTask(id, 'in_progress')}
            />
          ))}
        </div>
        
        <div className="column">
          <h2>In Progress ({tasks.inProgress.length})</h2>
          {tasks.inProgress.map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onMove={(id) => moveTask(id, 'done')}
            />
          ))}
        </div>
        
        <div className="column">
          <h2>Done ({tasks.done.length})</h2>
          {tasks.done.map(task => (
            <TaskCard key={task._id} task={task} />
          ))}
        </div>
      </div>
    </div>
  );
};

const TaskCard = ({ task, onMove }) => (
  <div className={`task-card priority-${task.priority}`}>
    <h4>{task.title}</h4>
    <p>{task.description}</p>
    <div className="task-meta">
      <span>Due: {new Date(task.dueDate).toLocaleDateString()}</span>
      {onMove && (
        <button onClick={() => onMove(task._id)}>Move →</button>
      )}
    </div>
  </div>
);

export default UserDashboard;
```

## Common Patterns

### Protected Routes

```javascript
// frontend/src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const token = localStorage.getItem('token');
  const userRole = localStorage.getItem('userRole');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (requireAdmin && userRole !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### Ticket Creation with AI Classification

```javascript
// frontend/src/components/CreateTicket.jsx
import React, { useState } from 'react';
import { mlService } from '../services/api';

const CreateTicket = () => {
  const [formData, setFormData] = useState({
    title: '',
    description: '',
    priority: 'medium'
  });
  const [classification, setClassification] = useState(null);

  const handleClassify = async () => {
    try {
      const { data } = await mlService.classifyTicket(formData);
      setClassification(data);
    } catch (error) {
      console.error('Classification error:', error);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    // First classify the ticket
    await handleClassify();
    
    // Then submit with classification data
    // await ticketService.createTicket({ ...formData, ...classification });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        placeholder="Ticket Title"
        value={formData.title}
        onChange={(e) => setFormData({ ...formData, title: e.target.value })}
        required
      />
      
      <textarea
        placeholder="Description"
        value={formData.description}
        onChange={(e) => setFormData({ ...formData, description: e.target.value })}
        required
      />
      
      <select
        value={formData.priority}
        onChange={(e) => setFormData({ ...formData, priority: e.target.value })}
      >
        <option value="low">Low</option>
        <option value="medium">Medium</option>
        <option value="high">High</option>
      </select>
      
      <button type="button" onClick={handleClassify}>
        Classify with AI
      </button>
      
      {classification && (
        <div className="classification-result">
          <p>Category: {classification.category}</p>
          <p>Department: {classification.department}</p>
          <p>Suggested Priority: {classification.priority}</p>
          <p>Est. Resolution: {classification.estimated_resolution_hours}h</p>
        </div>
      )}
      
      <button type="submit">Create Ticket</button>
    </form>
  );
};

export default CreateTicket;
```

## Configuration

### Environment Variables

**Backend (.env)**:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**:
```env
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
MODEL_PATH=./models
LOG_LEVEL=INFO
```

**Frontend (.env)**:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

### MongoDB Schema Models

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  status: { 
    type: String, 
    enum: ['todo', 'in_progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in minutes
  completedAt: Date,
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  category: String,
  department: String,
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  status: { 
    type: String, 
    enum: ['open', 'in_progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  aiClassification: {
    category: String,
    confidence
