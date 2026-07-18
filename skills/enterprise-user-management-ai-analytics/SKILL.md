---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management for enterprise applications
triggers:
  - "set up enterprise user management with AI"
  - "implement user task tracking with AI analytics"
  - "create admin dashboard with user management"
  - "build user management system with risk detection"
  - "integrate AI-powered ticket classification"
  - "develop task management with burnout detection"
  - "setup JWT authentication for user management"
  - "implement Kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack enterprise user management platform combining React frontend, Node.js backend, and FastAPI ML service. Features role-based access control, task management with Kanban boards, support ticket system, and AI-powered analytics including risk prediction, anomaly detection, burnout analysis, and predictive insights.

## Installation

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup (Node.js)

```bash
cd backend
npm install
```

Create `.env` file:
```env
PORT=5000
MONGODB_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
NODE_ENV=development
```

Start backend:
```bash
npm start
```

### ML Service Setup (FastAPI)

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:
```env
ML_MODEL_PATH=./models
API_HOST=0.0.0.0
API_PORT=8000
```

Start ML service:
```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install
```

Create `.env` file:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Core Architecture

### Backend API Structure (Node.js/Express)

**User Authentication & Authorization**

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'Authentication required' });
  }
  
  try {
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

**User Routes**

```javascript
// routes/userRoutes.js
const express = require('express');
const router = express.Router();
const { authMiddleware, adminOnly } = require('../middleware/auth');
const User = require('../models/User');

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    
    if (!user || !(await user.comparePassword(password))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, email: user.email, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );
    
    res.json({ token, user: { id: user._id, email: user.email, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get all users (Admin only)
router.get('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create user (Admin only)
router.post('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const user = new User(req.body);
    await user.save();
    res.status(201).json(user);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update user (Admin only)
router.put('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(req.params.id, req.body, { new: true });
    res.json(user);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Delete user (Admin only)
router.delete('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    await User.findByIdAndDelete(req.params.id);
    res.json({ message: 'User deleted' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

**Task Management Routes**

```javascript
// routes/taskRoutes.js
const express = require('express');
const router = express.Router();
const { authMiddleware } = require('../middleware/auth');
const Task = require('../models/Task');

// Get user tasks
router.get('/my-tasks', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
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
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Track time
router.post('/:id/time-log', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body;
    const task = await Task.findById(req.params.id);
    task.timeSpent = (task.timeSpent || 0) + duration;
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

**Support Ticket Routes**

```javascript
// routes/ticketRoutes.js
const express = require('express');
const router = express.Router();
const { authMiddleware } = require('../middleware/auth');
const axios = require('axios');
const Ticket = require('../models/Ticket');

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(`${process.env.ML_API_URL}/classify-ticket`, {
      title,
      description
    });
    
    const ticket = new Ticket({
      title,
      description,
      priority,
      userId: req.user.id,
      category: mlResponse.data.category,
      suggestedAssignment: mlResponse.data.suggested_team,
      aiConfidence: mlResponse.data.confidence
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ userId: req.user.id });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### MongoDB Models

**User Model**

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  firstName: { type: String, required: true },
  lastName: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  position: String,
  isActive: { type: Boolean, default: true },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now }
});

userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

userSchema.methods.comparePassword = async function(password) {
  return bcrypt.compare(password, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

**Task Model**

```javascript
// models/Task.js
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
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

**Ticket Model**

```javascript
// models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
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
  category: String,
  userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  suggestedAssignment: String,
  aiConfidence: Number,
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## ML Service (FastAPI)

### AI Classification & Analytics

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from river import anomaly, linear_model
import joblib
import os

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
ticket_classifier = None
vectorizer = None
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)
burnout_model = linear_model.LogisticRegression()

class TicketRequest(BaseModel):
    title: str
    description: str

class RiskAnalysisRequest(BaseModel):
    user_id: str
    failed_logins: int
    unusual_activity_count: int
    access_violations: int
    login_times: List[int]  # hours of day

class BurnoutRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_task_duration: float
    overtime_hours: float
    days_without_break: int

class ProjectPredictionRequest(BaseModel):
    total_tasks: int
    completed_tasks: int
    days_elapsed: int
    team_size: int
    complexity_score: float

@app.on_event("startup")
async def load_models():
    global ticket_classifier, vectorizer
    try:
        model_path = os.getenv('ML_MODEL_PATH', './models')
        vectorizer = joblib.load(f'{model_path}/vectorizer.pkl')
        ticket_classifier = joblib.load(f'{model_path}/ticket_classifier.pkl')
    except:
        # Initialize with dummy data if models don't exist
        vectorizer = TfidfVectorizer(max_features=100)
        ticket_classifier = MultinomialNB()

@app.post("/classify-ticket")
async def classify_ticket(request: TicketRequest):
    try:
        text = f"{request.title} {request.description}"
        features = vectorizer.transform([text])
        
        if hasattr(ticket_classifier, 'predict'):
            category = ticket_classifier.predict(features)[0]
            confidence = float(max(ticket_classifier.predict_proba(features)[0]))
        else:
            # Fallback classification
            category = "general"
            confidence = 0.5
        
        # Map category to team
        team_mapping = {
            "technical": "Engineering",
            "billing": "Finance",
            "account": "Customer Success",
            "general": "Support"
        }
        
        return {
            "category": category,
            "suggested_team": team_mapping.get(category, "Support"),
            "confidence": confidence
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-risk")
async def analyze_risk(request: RiskAnalysisRequest):
    try:
        # Feature engineering
        risk_score = 0
        
        # Failed login analysis
        if request.failed_logins > 5:
            risk_score += 30
        elif request.failed_logins > 2:
            risk_score += 15
        
        # Unusual activity
        risk_score += min(request.unusual_activity_count * 10, 30)
        
        # Access violations
        risk_score += min(request.access_violations * 15, 40)
        
        # Login time anomalies (e.g., 2am-5am)
        unusual_hours = sum(1 for hour in request.login_times if hour < 6 or hour > 23)
        if unusual_hours > len(request.login_times) * 0.3:
            risk_score += 20
        
        risk_score = min(risk_score, 100)
        
        # Determine risk level
        if risk_score >= 70:
            level = "critical"
            action = "Immediate review required"
        elif risk_score >= 50:
            level = "high"
            action = "Enhanced monitoring"
        elif risk_score >= 30:
            level = "medium"
            action = "Standard monitoring"
        else:
            level = "low"
            action = "No action required"
        
        return {
            "user_id": request.user_id,
            "risk_score": risk_score,
            "risk_level": level,
            "recommended_action": action,
            "factors": {
                "failed_logins": request.failed_logins,
                "unusual_activity": request.unusual_activity_count,
                "access_violations": request.access_violations,
                "unusual_login_times": unusual_hours
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(features: dict):
    try:
        # Convert features to numeric array
        feature_vector = {
            'login_frequency': features.get('login_frequency', 0),
            'data_access_volume': features.get('data_access_volume', 0),
            'api_calls': features.get('api_calls', 0),
            'error_rate': features.get('error_rate', 0)
        }
        
        # Update anomaly detector
        score = anomaly_detector.score_one(feature_vector)
        anomaly_detector.learn_one(feature_vector)
        
        is_anomaly = score > 0.7
        
        return {
            "is_anomaly": is_anomaly,
            "anomaly_score": float(score),
            "threshold": 0.7,
            "message": "Anomalous behavior detected" if is_anomaly else "Normal behavior"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-burnout")
async def predict_burnout(request: BurnoutRequest):
    try:
        # Calculate burnout indicators
        completion_rate = request.tasks_completed / max(request.tasks_assigned, 1)
        workload_score = request.tasks_assigned / max(request.team_size, 1) if hasattr(request, 'team_size') else request.tasks_assigned
        
        burnout_score = 0
        
        # Low completion rate
        if completion_rate < 0.5:
            burnout_score += 25
        
        # High workload
        if workload_score > 10:
            burnout_score += 30
        
        # Overtime
        if request.overtime_hours > 20:
            burnout_score += 25
        elif request.overtime_hours > 10:
            burnout_score += 15
        
        # No breaks
        if request.days_without_break > 14:
            burnout_score += 20
        elif request.days_without_break > 7:
            burnout_score += 10
        
        burnout_score = min(burnout_score, 100)
        
        if burnout_score >= 70:
            risk_level = "critical"
            recommendation = "Immediate intervention required - reduce workload"
        elif burnout_score >= 50:
            risk_level = "high"
            recommendation = "Schedule time off and redistribute tasks"
        elif burnout_score >= 30:
            risk_level = "medium"
            recommendation = "Monitor workload and ensure regular breaks"
        else:
            risk_level = "low"
            recommendation = "Maintain current pace"
        
        return {
            "user_id": request.user_id,
            "burnout_score": burnout_score,
            "risk_level": risk_level,
            "recommendation": recommendation,
            "metrics": {
                "completion_rate": completion_rate,
                "overtime_hours": request.overtime_hours,
                "days_without_break": request.days_without_break
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-project-delay")
async def predict_project_delay(request: ProjectPredictionRequest):
    try:
        # Calculate completion velocity
        completion_rate = request.completed_tasks / max(request.total_tasks, 1)
        days_per_task = request.days_elapsed / max(request.completed_tasks, 1)
        remaining_tasks = request.total_tasks - request.completed_tasks
        
        estimated_days_remaining = remaining_tasks * days_per_task
        
        # Adjust for team size and complexity
        team_factor = 1 / max(request.team_size, 1)
        complexity_factor = 1 + (request.complexity_score / 10)
        
        adjusted_days = estimated_days_remaining * team_factor * complexity_factor
        
        # Determine delay risk
        if adjusted_days > 60:
            delay_risk = "high"
            message = "Project likely to experience significant delays"
        elif adjusted_days > 30:
            delay_risk = "medium"
            message = "Project may experience moderate delays"
        else:
            delay_risk = "low"
            message = "Project on track for timely completion"
        
        return {
            "completion_percentage": completion_rate * 100,
            "estimated_days_remaining": int(adjusted_days),
            "delay_risk": delay_risk,
            "message": message,
            "recommendations": [
                "Increase team size" if request.team_size < 3 else None,
                "Reduce complexity" if request.complexity_score > 7 else None,
                "Prioritize critical tasks" if completion_rate < 0.3 else None
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Frontend Integration (React)

### API Service Layer

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

// Create axios instances
const apiClient = axios.create({
  baseURL: API_URL,
  headers: { 'Content-Type': 'application/json' }
});

const mlClient = axios.create({
  baseURL: ML_API_URL,
  headers: { 'Content-Type': 'application/json' }
});

// Add token to requests
apiClient.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authService = {
  login: (email, password) => 
    apiClient.post('/api/users/login', { email, password }),
  
  getCurrentUser: () => 
    apiClient.get('/api/users/me')
};

export const userService = {
  getAll: () => apiClient.get('/api/users'),
  create: (data) => apiClient.post('/api/users', data),
  update: (id, data) => apiClient.put(`/api/users/${id}`, data),
  delete: (id) => apiClient.delete(`/api/users/${id}`)
};

export const taskService = {
  getMyTasks: () => apiClient.get('/api/tasks/my-tasks'),
  create: (data) => apiClient.post('/api/tasks', data),
  updateStatus: (id, status) => 
    apiClient.patch(`/api/tasks/${id}/status`, { status }),
  logTime: (id, duration) => 
    apiClient.post(`/api/tasks/${id}/time-log`, { duration })
};

export const ticketService = {
  create: (data) => apiClient.post('/api/tickets', data),
  getMyTickets: () => apiClient.get('/api/tickets/my-tickets')
};

export const mlService = {
  analyzeRisk: (data) => mlClient.post('/analyze-risk', data),
  predictBurnout: (data) => mlClient.post('/predict-burnout', data),
  detectAnomaly: (data) => mlClient.post('/detect-anomaly', data),
  predictProjectDelay: (data) => mlClient.post('/predict-project-delay', data)
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    try {
      const response = await taskService.getMyTasks();
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        'in-progress': response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error loading tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await taskService.updateStatus(taskId, newStatus);
      loadTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-badge ${task.priority}`}>
        {task.priority}
      </span>
      <div className="task-actions">
        <select 
          value={task.status} 
          onChange={(e) => moveTask(task._id, e.target.value)}
        >
          <option value="todo">To Do</option>
          <option value="in-progress">In Progress</option>
          <option value="done">Done</option>
        </select>
      </div>
    </div>
  );

  if (loading) return <div>Loading...</div>;

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>In Progress ({tasks['in-progress'].length})</h3>
        {tasks['in-progress'].map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
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
// frontend/src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import { mlService } from '../services/api';

const AIAnalyticsDashboard = ({ userId, userData }) => {
  const [riskAnalysis, setRiskAnalysis] = useState(null);
  const [burnoutPrediction, setBurnoutPrediction] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadAnalytics();
  }, [userId]);

  const loadAnalytics = async () => {
    try {
      // Risk analysis
      const riskData = await mlService.analyzeRisk({
        user_id: userId,
        failed_logins: userData.failedLogins || 0,
        unusual_activity_count: userData.unusualActivityCount || 0,
        access_violations: userData.accessViolations || 0,
        login_times: userData.recentLoginHours || []
      });
      setRiskAnalysis(riskData.data);

      // Burnout prediction
      const burnoutData = await mlService.predictBurnout({
        user_id: userId,
        tasks_assigned: userData.tasksAssigned || 0,
        tasks_completed: userData.tasksCompleted || 0,
        avg_task_duration: userData.avgTaskDuration || 0,
        overtime_hours: userData.overtimeHours || 0,
        days_without_break: userData.daysWithoutBreak || 0
      });
      setBurnoutPrediction(burnoutData.data);
    } catch (error) {
      console.error('Error loading analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics-dashboard">
      <div className="analytics-card">
        <h3>Risk Analysis</h3>
        <div className={`risk-indicator ${riskAnalysis?.risk_level}`}>
          <div className="risk-score">{riskAnalysis?.risk_score}/100</div>
          <div className="risk-level">{riskAnalysis?.risk_level}</div>
        </div>
        <p>{riskAnalysis?.recommended_action}</p>
      </div>

      <div className="analytics-card">
        <h3>Burnout Prediction</h3>
        <div className={`burnout-indicator ${burnoutPrediction?.risk_level}`}>
          <div className="burnout-score">{burnoutPrediction?.burnout_score}/100</div>
          <div className="risk-level">{burnoutPrediction?.risk_level}</div>
        </div>
        <p>{burnoutPrediction?.recommendation}</p>
        <div className="metrics">
          <span>Completion Rate: {(burnoutPrediction?.metrics?.completion_rate * 100).toFixed(1)}%</span>
          <span>Overtime: {burnoutPrediction?.metrics?.overtime_hours}h</span>
        </div>
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Common Patterns

### User Authentication Flow

```javascript
// Login and store JWT
