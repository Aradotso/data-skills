---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create admin dashboard with task tracking"
  - "integrate ML service for risk prediction"
  - "build user management with ticket system"
  - "deploy enterprise system with AI features"
  - "configure user roles and authentication"
  - "add AI-powered anomaly detection"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System is a full-stack JavaScript application that combines user/task management with AI-powered analytics. It provides:

- **User Management**: CRUD operations with role-based access control (Admin/User)
- **Task Tracking**: Kanban boards with time tracking and status management
- **Support Tickets**: AI-classified ticket routing and resolution tracking
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project insights
- **Real-time Monitoring**: Audit logs, performance metrics, and security alerts

The system consists of three main components:
1. **Frontend** (React.js) - User interface and dashboards
2. **Backend** (Node.js/Express) - REST API and business logic
3. **ML Service** (FastAPI/Python) - AI/ML models for analytics

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
```

### Full Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env
# Edit .env with your MongoDB URI and JWT secret
npm start  # Runs on http://localhost:5000

# ML Service setup (separate terminal)
cd ../ml-service
pip install -r requirements.txt
cp .env.example .env
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup (separate terminal)
cd ../frontend
npm install
cp .env.example .env
# Configure API endpoints in .env
npm start  # Runs on http://localhost:3000
```

## Environment Configuration

### Backend (.env)

```bash
# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
DB_NAME=enterprise_ums

# Authentication
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d

# Server
PORT=5000
NODE_ENV=development

# ML Service
ML_SERVICE_URL=http://localhost:8000
```

### Frontend (.env)

```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_JWT_KEY=your_jwt_secret_key_here
```

### ML Service (.env)

```bash
# API Configuration
API_HOST=0.0.0.0
API_PORT=8000

# Model paths
MODEL_DIR=./models
RISK_MODEL_PATH=./models/risk_model.pkl
ANOMALY_MODEL_PATH=./models/anomaly_model.pkl

# Backend integration
BACKEND_URL=http://localhost:5000
```

## Key API Endpoints

### Authentication

```javascript
// Backend: POST /api/auth/login
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  const data = await response.json();
  return data.token; // JWT token
};

// Backend: POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return await response.json();
};
```

### User Management (Admin)

```javascript
// Backend: GET /api/users (Admin only)
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return await response.json();
};

// Backend: PUT /api/users/:id
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};

// Backend: DELETE /api/users/:id
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### Task Management

```javascript
// Backend: GET /api/tasks
const getTasks = async (token, userId = null) => {
  const url = userId 
    ? `http://localhost:5000/api/tasks?userId=${userId}`
    : 'http://localhost:5000/api/tasks';
  
  const response = await fetch(url, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Backend: POST /api/tasks
const createTask = async (taskData, token) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'inprogress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return await response.json();
};

// Backend: PUT /api/tasks/:id/status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'todo', 'inprogress', 'done'
  });
  return await response.json();
};

// Backend: POST /api/tasks/:id/time
const trackTime = async (taskId, timeSpent, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ timeSpent }) // in seconds
  });
  return await response.json();
};
```

### Support Tickets

```javascript
// Backend: POST /api/tickets
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority, // 'low', 'medium', 'high', 'critical'
      category: ticketData.category // 'technical', 'billing', 'general'
    })
  });
  return await response.json();
};

// Backend: GET /api/tickets
const getTickets = async (token, filters = {}) => {
  const queryParams = new URLSearchParams(filters).toString();
  const response = await fetch(`http://localhost:5000/api/tickets?${queryParams}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Backend: PUT /api/tickets/:id/assign
const assignTicket = async (ticketId, assigneeId, token) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}/assign`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ assignedTo: assigneeId })
  });
  return await response.json();
};
```

## AI/ML Service Integration

### Risk Prediction

```javascript
// ML Service: POST /api/ml/predict-risk
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      features: {
        task_completion_rate: 0.75,
        average_response_time: 120, // minutes
        missed_deadlines: 3,
        overdue_tasks: 2,
        active_tasks: 8
      }
    })
  });
  const data = await response.json();
  return data.risk_score; // 0-1 scale
};

// Python ML Service implementation
"""
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np

app = FastAPI()

class RiskPredictionRequest(BaseModel):
    user_id: str
    features: dict

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    model = joblib.load('models/risk_model.pkl')
    
    feature_vector = [
        request.features.get('task_completion_rate', 0),
        request.features.get('average_response_time', 0),
        request.features.get('missed_deadlines', 0),
        request.features.get('overdue_tasks', 0),
        request.features.get('active_tasks', 0)
    ]
    
    risk_score = model.predict_proba([feature_vector])[0][1]
    
    return {
        "user_id": request.user_id,
        "risk_score": float(risk_score),
        "risk_level": "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
    }
"""
```

### Anomaly Detection

```javascript
// ML Service: POST /api/ml/detect-anomaly
const detectAnomaly = async (userActivity) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userActivity.userId,
      login_time: userActivity.loginTime,
      location: userActivity.location,
      device: userActivity.device,
      actions_count: userActivity.actionsCount,
      data_accessed: userActivity.dataAccessed
    })
  });
  return await response.json();
};

// Python implementation
"""
from river import anomaly
from datetime import datetime

anomaly_detector = anomaly.HalfSpaceTrees()

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(activity: dict):
    features = {
        'hour': datetime.fromisoformat(activity['login_time']).hour,
        'actions_count': activity['actions_count'],
        'data_accessed': activity['data_accessed'],
        'is_unusual_location': activity.get('is_unusual_location', 0)
    }
    
    score = anomaly_detector.score_one(features)
    anomaly_detector.learn_one(features)
    
    is_anomaly = score > 0.7
    
    return {
        "user_id": activity['user_id'],
        "anomaly_score": float(score),
        "is_anomaly": is_anomaly,
        "timestamp": activity['login_time']
    }
"""
```

### Burnout Analysis

```javascript
// ML Service: POST /api/ml/burnout-analysis
const analyzeBurnout = async (userId, workloadData) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-analysis', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      weekly_hours: workloadData.weeklyHours,
      consecutive_work_days: workloadData.consecutiveWorkDays,
      overtime_hours: workloadData.overtimeHours,
      task_load: workloadData.taskLoad,
      stress_indicators: workloadData.stressIndicators
    })
  });
  return await response.json();
};

// Python implementation
"""
@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(data: dict):
    burnout_score = calculate_burnout_score(
        data['weekly_hours'],
        data['consecutive_work_days'],
        data['overtime_hours'],
        data['task_load']
    )
    
    recommendations = []
    if burnout_score > 0.7:
        recommendations = [
            "Reduce task load immediately",
            "Schedule mandatory time off",
            "Redistribute tasks to team"
        ]
    elif burnout_score > 0.5:
        recommendations = [
            "Monitor workload closely",
            "Consider task delegation",
            "Encourage breaks"
        ]
    
    return {
        "user_id": data['user_id'],
        "burnout_score": float(burnout_score),
        "risk_level": "critical" if burnout_score > 0.7 else "warning" if burnout_score > 0.5 else "normal",
        "recommendations": recommendations
    }

def calculate_burnout_score(hours, days, overtime, load):
    normalized_hours = min(hours / 60, 1.0)
    normalized_days = min(days / 14, 1.0)
    normalized_overtime = min(overtime / 20, 1.0)
    normalized_load = min(load / 20, 1.0)
    
    return (normalized_hours * 0.3 + normalized_days * 0.25 + 
            normalized_overtime * 0.25 + normalized_load * 0.2)
"""
```

### AI Ticket Classification

```javascript
// ML Service: POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketText.title,
      description: ticketText.description
    })
  });
  return await response.json();
};

// Python implementation with sklearn
"""
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib

@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: dict):
    vectorizer = joblib.load('models/tfidf_vectorizer.pkl')
    classifier = joblib.load('models/ticket_classifier.pkl')
    
    text = f"{ticket['title']} {ticket['description']}"
    text_vector = vectorizer.transform([text])
    
    category = classifier.predict(text_vector)[0]
    confidence = max(classifier.predict_proba(text_vector)[0])
    
    priority_score = calculate_priority(ticket['description'])
    
    return {
        "category": category,  # 'technical', 'billing', 'general', 'urgent'
        "confidence": float(confidence),
        "suggested_priority": priority_score,
        "routing": get_department_routing(category)
    }

def calculate_priority(text):
    urgent_keywords = ['urgent', 'critical', 'down', 'not working', 'asap']
    text_lower = text.lower()
    urgency_score = sum(1 for keyword in urgent_keywords if keyword in text_lower)
    return min(urgency_score / len(urgent_keywords), 1.0)

def get_department_routing(category):
    routing_map = {
        'technical': 'IT Support',
        'billing': 'Finance',
        'general': 'Customer Service',
        'urgent': 'Priority Queue'
    }
    return routing_map.get(category, 'General Queue')
"""
```

## Common Patterns

### React Component: User Dashboard

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [tickets, setTickets] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      // Fetch tasks
      const tasksRes = await fetch('http://localhost:5000/api/tasks/my-tasks', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const tasksData = await tasksRes.json();
      setTasks(tasksData);

      // Fetch tickets
      const ticketsRes = await fetch('http://localhost:5000/api/tickets/my-tickets', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const ticketsData = await ticketsRes.json();
      setTickets(ticketsData);

      // Fetch AI analytics
      const analyticsRes = await fetch('http://localhost:8000/api/ml/user-analytics', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: getUserId() })
      });
      const analyticsData = await analyticsRes.json();
      setAnalytics(analyticsData);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
        method: 'PUT',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ status: newStatus })
      });
      fetchDashboardData(); // Refresh
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const getUserId = () => {
    const payload = JSON.parse(atob(token.split('.')[1]));
    return payload.userId;
  };

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {/* AI Analytics Summary */}
      {analytics && (
        <div className="analytics-card">
          <h2>Performance Insights</h2>
          <div className="metric">
            <span>Risk Level:</span>
            <span className={`badge ${analytics.risk_level}`}>
              {analytics.risk_level}
            </span>
          </div>
          {analytics.burnout_warning && (
            <div className="warning">
              ⚠️ Burnout risk detected. Consider reducing workload.
            </div>
          )}
        </div>
      )}

      {/* Kanban Board */}
      <div className="kanban-board">
        <div className="kanban-column">
          <h3>To Do</h3>
          {tasks.filter(t => t.status === 'todo').map(task => (
            <TaskCard 
              key={task._id} 
              task={task}
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
        <div className="kanban-column">
          <h3>In Progress</h3>
          {tasks.filter(t => t.status === 'inprogress').map(task => (
            <TaskCard 
              key={task._id} 
              task={task}
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
        <div className="kanban-column">
          <h3>Done</h3>
          {tasks.filter(t => t.status === 'done').map(task => (
            <TaskCard 
              key={task._id} 
              task={task}
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
      </div>

      {/* Tickets */}
      <div className="tickets-section">
        <h2>My Tickets</h2>
        {tickets.map(ticket => (
          <TicketCard key={ticket._id} ticket={ticket} />
        ))}
      </div>
    </div>
  );
};

const TaskCard = ({ task, onStatusChange }) => {
  return (
    <div className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-footer">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        <select 
          value={task.status}
          onChange={(e) => onStatusChange(task._id, e.target.value)}
        >
          <option value="todo">To Do</option>
          <option value="inprogress">In Progress</option>
          <option value="done">Done</option>
        </select>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Backend: Task Controller with AI Integration

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      createdBy: req.user.id,
      status: 'todo'
    });

    await task.save();

    // Trigger AI analysis for workload
    try {
      await axios.post(`${process.env.ML_SERVICE_URL}/api/ml/analyze-workload`, {
        user_id: assignedTo,
        new_task_priority: priority
      });
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
      // Continue even if ML service fails
    }

    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });

    // Get AI insights for user
    let aiInsights = null;
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/ml/task-insights`,
        { user_id: req.user.id, tasks }
      );
      aiInsights = mlResponse.data;
    } catch (mlError) {
      console.error('ML insights error:', mlError.message);
    }

    res.json({
      tasks,
      insights: aiInsights
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }

    await task.save();

    // Update ML models with completion data
    if (status === 'done') {
      try {
        await axios.post(`${process.env.ML_SERVICE_URL}/api/ml/task-completed`, {
          user_id: task.assignedTo,
          task_id: task._id,
          completion_time: (new Date() - task.createdAt) / 1000 / 60 // minutes
        });
      } catch (mlError) {
        console.error('ML update error:', mlError.message);
      }
    }

    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const { timeSpent } = req.body; // in seconds
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.timeTracked = (task.timeTracked || 0) + timeSpent;
    await task.save();

    res.json({ timeTracked: task.timeTracked });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

### MongoDB Schema Examples

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
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

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);

// backend/models/Task.js
const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
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
  status: {
    type: String,
    enum: ['todo', 'inprogress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  dueDate: Date,
  completedAt: Date,
  timeTracked: {
    type: Number,
    default: 0 // in seconds
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', taskSchema);

// backend/models/Ticket.js
const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
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
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'urgent'],
    default: 'general'
  },
  aiClassification: {
    suggestedCategory: String,
    confidence: Number,
    suggestedPriority: String
  },
  resolvedAt: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Admin Dashboard Pattern

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [systemAnalytics, setSystemAnalytics] = useState(null);
  const [alerts, setAlerts] = useState([]);
  const token = localStorage.getItem('token');

  const api = axios.create({
    baseURL: process.env.REACT_APP_API_URL,
    headers: { 'Authorization': `Bearer ${token}` }
  });

  useEffect(() => {
    fetchAdminData();
  }, []);

  const fetchAdminData = async () => {
    try {
      // Fetch all users
      const usersRes = await api.get('/users');
      setUsers(usersRes.data);

      // Fetch system-wide analytics from ML service
      const analyticsRes = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/system-analytics`,
        { user_ids: usersRes.data.map(u => u._id) }
      );
      setSystemAnalytics(
