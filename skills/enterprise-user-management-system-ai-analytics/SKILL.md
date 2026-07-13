---
name: enterprise-user-management-system-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics into user management"
  - "implement JWT authentication for user management"
  - "create admin dashboard with user analytics"
  - "add AI-based ticket classification system"
  - "build kanban board for task management"
  - "detect user burnout with ML models"
  - "implement role-based access control with AI"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user management, task tracking, and support ticket systems with AI-powered insights. It provides risk detection, anomaly detection, burnout analysis, and predictive project insights using machine learning models built with FastAPI, scikit-learn, and River.

The system consists of three main components:
- **Frontend**: React.js application with user/admin dashboards
- **Backend**: Node.js REST API with MongoDB and JWT authentication
- **ML Service**: FastAPI service for AI/ML predictions and analytics

## Installation

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

## Core Architecture

### Backend API Structure (Node.js)

```javascript
// server.js - Main entry point
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const jwt = require('jsonwebtoken');
require('dotenv').config();

const app = express();

app.use(cors());
app.use(express.json());

// Connect to MongoDB
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => console.log('MongoDB Connected'))
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

### Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

module.exports = function(req, res, next) {
  const token = req.header('x-auth-token');
  
  if (!token) {
    return res.status(401).json({ msg: 'No token, authorization denied' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded.user;
    next();
  } catch (err) {
    res.status(401).json({ msg: 'Token is not valid' });
  }
};

// middleware/admin.js
module.exports = function(req, res, next) {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ msg: 'Access denied. Admin only.' });
  }
  next();
};
```

### User Model

```javascript
// models/User.js
const mongoose = require('mongoose');

const UserSchema = new mongoose.Schema({
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
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  tasksAssigned: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Task'
  }],
  workloadScore: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('User', UserSchema);
```

### Task Model

```javascript
// models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
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
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  dueDate: Date,
  timeTracked: {
    type: Number,
    default: 0
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', TaskSchema);
```

### Ticket Model

```javascript
// models/Ticket.js
const mongoose = require('mongoose');

const TicketSchema = new mongoose.Schema({
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
    enum: ['technical', 'administrative', 'hr', 'other']
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical']
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
  aiClassified: {
    type: Boolean,
    default: false
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Ticket', TicketSchema);
```

## ML Service API

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
from river import linear_model, metrics
import joblib
import os

app = FastAPI(title="Enterprise AI Analytics Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Risk detection model
risk_model = RandomForestClassifier(n_estimators=100, random_state=42)

# Anomaly detection model
anomaly_model = IsolationForest(contamination=0.1, random_state=42)

# Online learning model for burnout detection
burnout_model = linear_model.LogisticRegression()

class UserBehavior(BaseModel):
    user_id: str
    login_frequency: float
    task_completion_rate: float
    average_task_time: float
    missed_deadlines: int
    workload_score: float
    overtime_hours: float

class TicketData(BaseModel):
    title: str
    description: str

class PredictionResponse(BaseModel):
    prediction: str
    confidence: float
    risk_score: Optional[float] = None

@app.get("/")
def read_root():
    return {"status": "AI Analytics Service Running"}

@app.post("/api/ml/risk-detection", response_model=PredictionResponse)
async def detect_risk(data: UserBehavior):
    """Predict user risk level based on behavior patterns"""
    try:
        features = np.array([[
            data.login_frequency,
            data.task_completion_rate,
            data.average_task_time,
            data.missed_deadlines,
            data.workload_score,
            data.overtime_hours
        ]])
        
        # Simple rule-based risk scoring
        risk_score = (
            (1 - data.task_completion_rate) * 30 +
            data.missed_deadlines * 15 +
            (data.workload_score / 10) * 25 +
            (data.overtime_hours / 40) * 30
        )
        
        if risk_score > 70:
            prediction = "high"
        elif risk_score > 40:
            prediction = "medium"
        else:
            prediction = "low"
        
        confidence = min(abs(risk_score - 50) / 50, 1.0)
        
        return PredictionResponse(
            prediction=prediction,
            confidence=confidence,
            risk_score=risk_score
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(data: UserBehavior):
    """Detect anomalous user behavior"""
    try:
        features = np.array([[
            data.login_frequency,
            data.task_completion_rate,
            data.average_task_time,
            data.missed_deadlines,
            data.workload_score,
            data.overtime_hours
        ]])
        
        # Check for anomalies
        is_anomaly = (
            data.login_frequency > 50 or
            data.login_frequency < 1 or
            data.task_completion_rate < 0.3 or
            data.overtime_hours > 60 or
            data.missed_deadlines > 5
        )
        
        return {
            "is_anomaly": is_anomaly,
            "anomaly_score": float(data.workload_score) if is_anomaly else 0.0,
            "factors": {
                "unusual_login": data.login_frequency > 50,
                "low_completion": data.task_completion_rate < 0.3,
                "excessive_overtime": data.overtime_hours > 60,
                "many_missed_deadlines": data.missed_deadlines > 5
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection", response_model=PredictionResponse)
async def detect_burnout(data: UserBehavior):
    """Detect employee burnout risk"""
    try:
        burnout_score = (
            (data.workload_score / 10) * 35 +
            (data.overtime_hours / 40) * 30 +
            (1 - data.task_completion_rate) * 20 +
            (data.missed_deadlines / 10) * 15
        )
        
        if burnout_score > 70:
            prediction = "high_risk"
            recommendation = "Immediate workload reduction recommended"
        elif burnout_score > 45:
            prediction = "moderate_risk"
            recommendation = "Monitor closely and consider workload adjustment"
        else:
            prediction = "low_risk"
            recommendation = "Normal workload management"
        
        return PredictionResponse(
            prediction=prediction,
            confidence=min(burnout_score / 100, 1.0),
            risk_score=burnout_score
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketData):
    """Classify and route support tickets using AI"""
    try:
        text = f"{ticket.title} {ticket.description}".lower()
        
        # Simple keyword-based classification
        if any(word in text for word in ['bug', 'error', 'crash', 'not working']):
            category = 'technical'
            priority = 'high'
        elif any(word in text for word in ['payroll', 'leave', 'salary', 'hr']):
            category = 'hr'
            priority = 'medium'
        elif any(word in text for word in ['access', 'permission', 'account']):
            category = 'administrative'
            priority = 'medium'
        else:
            category = 'other'
            priority = 'low'
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85,
            "suggested_department": category.title()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/project-insights")
async def get_project_insights(data: dict):
    """Generate predictive project insights"""
    try:
        tasks = data.get('tasks', [])
        total_tasks = len(tasks)
        completed_tasks = sum(1 for t in tasks if t.get('status') == 'done')
        overdue_tasks = sum(1 for t in tasks if t.get('isOverdue', False))
        
        completion_rate = completed_tasks / total_tasks if total_tasks > 0 else 0
        
        # Predict project delay risk
        delay_risk = overdue_tasks / total_tasks if total_tasks > 0 else 0
        
        insights = {
            "completion_rate": completion_rate,
            "delay_risk": delay_risk,
            "predicted_delay_days": int(delay_risk * 14),
            "recommendations": []
        }
        
        if delay_risk > 0.3:
            insights["recommendations"].append("High delay risk detected. Consider reallocating resources.")
        if completion_rate < 0.5:
            insights["recommendations"].append("Low completion rate. Review task priorities.")
        
        return insights
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend Integration

### API Service Helper

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

// Set auth token
export const setAuthToken = (token) => {
  if (token) {
    axios.defaults.headers.common['x-auth-token'] = token;
    localStorage.setItem('token', token);
  } else {
    delete axios.defaults.headers.common['x-auth-token'];
    localStorage.removeItem('token');
  }
};

// Auth APIs
export const login = async (email, password) => {
  const response = await axios.post(`${API_URL}/api/auth/login`, {
    email,
    password
  });
  setAuthToken(response.data.token);
  return response.data;
};

export const register = async (userData) => {
  const response = await axios.post(`${API_URL}/api/auth/register`, userData);
  return response.data;
};

// User APIs
export const getUsers = async () => {
  const response = await axios.get(`${API_URL}/api/users`);
  return response.data;
};

export const createUser = async (userData) => {
  const response = await axios.post(`${API_URL}/api/users`, userData);
  return response.data;
};

export const updateUser = async (userId, userData) => {
  const response = await axios.put(`${API_URL}/api/users/${userId}`, userData);
  return response.data;
};

export const deleteUser = async (userId) => {
  const response = await axios.delete(`${API_URL}/api/users/${userId}`);
  return response.data;
};

// Task APIs
export const getTasks = async () => {
  const response = await axios.get(`${API_URL}/api/tasks`);
  return response.data;
};

export const createTask = async (taskData) => {
  const response = await axios.post(`${API_URL}/api/tasks`, taskData);
  return response.data;
};

export const updateTask = async (taskId, taskData) => {
  const response = await axios.put(`${API_URL}/api/tasks/${taskId}`, taskData);
  return response.data;
};

// Ticket APIs
export const getTickets = async () => {
  const response = await axios.get(`${API_URL}/api/tickets`);
  return response.data;
};

export const createTicket = async (ticketData) => {
  const response = await axios.post(`${API_URL}/api/tickets`, ticketData);
  
  // Auto-classify with AI
  const classification = await classifyTicket({
    title: ticketData.title,
    description: ticketData.description
  });
  
  return { ...response.data, classification };
};

// ML Service APIs
export const getRiskPrediction = async (userBehavior) => {
  const response = await axios.post(`${ML_API_URL}/api/ml/risk-detection`, userBehavior);
  return response.data;
};

export const detectAnomaly = async (userBehavior) => {
  const response = await axios.post(`${ML_API_URL}/api/ml/anomaly-detection`, userBehavior);
  return response.data;
};

export const detectBurnout = async (userBehavior) => {
  const response = await axios.post(`${ML_API_URL}/api/ml/burnout-detection`, userBehavior);
  return response.data;
};

export const classifyTicket = async (ticketData) => {
  const response = await axios.post(`${ML_API_URL}/api/ml/classify-ticket`, ticketData);
  return response.data;
};

export const getProjectInsights = async (projectData) => {
  const response = await axios.post(`${ML_API_URL}/api/ml/project-insights`, projectData);
  return response.data;
};
```

### React Component Example - Admin Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { getUsers, getTasks, getProjectInsights, getRiskPrediction } from '../services/api';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [tasks, setTasks] = useState([]);
  const [insights, setInsights] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      const usersData = await getUsers();
      const tasksData = await getTasks();
      
      setUsers(usersData);
      setTasks(tasksData);

      // Get project insights
      const projectInsights = await getProjectInsights({ tasks: tasksData });
      setInsights(projectInsights);

      // Check risk for each user
      const risks = await Promise.all(
        usersData.map(async (user) => {
          const userTasks = tasksData.filter(t => t.assignedTo === user._id);
          const completedTasks = userTasks.filter(t => t.status === 'done').length;
          const missedDeadlines = userTasks.filter(t => t.isOverdue).length;

          const riskData = await getRiskPrediction({
            user_id: user._id,
            login_frequency: user.loginFrequency || 10,
            task_completion_rate: completedTasks / userTasks.length || 0,
            average_task_time: user.avgTaskTime || 120,
            missed_deadlines: missedDeadlines,
            workload_score: user.workloadScore || 50,
            overtime_hours: user.overtimeHours || 0
          });

          return { ...user, risk: riskData };
        })
      );

      const highRiskUsers = risks.filter(u => u.risk.prediction === 'high');
      setRiskUsers(highRiskUsers);
    } catch (error) {
      console.error('Error loading dashboard:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>

      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p>{users.length}</p>
        </div>
        <div className="stat-card">
          <h3>Total Tasks</h3>
          <p>{tasks.length}</p>
        </div>
        <div className="stat-card">
          <h3>High Risk Users</h3>
          <p className="risk-alert">{riskUsers.length}</p>
        </div>
        {insights && (
          <div className="stat-card">
            <h3>Project Completion</h3>
            <p>{(insights.completion_rate * 100).toFixed(1)}%</p>
          </div>
        )}
      </div>

      {insights && insights.recommendations.length > 0 && (
        <div className="recommendations">
          <h2>AI Recommendations</h2>
          <ul>
            {insights.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      )}

      {riskUsers.length > 0 && (
        <div className="risk-users">
          <h2>Users Requiring Attention</h2>
          <table>
            <thead>
              <tr>
                <th>Name</th>
                <th>Department</th>
                <th>Risk Level</th>
                <th>Risk Score</th>
                <th>Actions</th>
              </tr>
            </thead>
            <tbody>
              {riskUsers.map(user => (
                <tr key={user._id}>
                  <td>{user.name}</td>
                  <td>{user.department}</td>
                  <td className={`risk-${user.risk.prediction}`}>
                    {user.risk.prediction}
                  </td>
                  <td>{user.risk.risk_score?.toFixed(2)}</td>
                  <td>
                    <button onClick={() => viewUserDetails(user._id)}>
                      View Details
                    </button>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      )}
    </div>
  );
};

export default AdminDashboard;
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { DragDropContext, Droppable, Draggable } from 'react-beautiful-dnd';
import { getTasks, updateTask } from '../services/api';

const KanbanBoard = ({ userId }) => {
  const [columns, setColumns] = useState({
    todo: { name: 'To Do', items: [] },
    'in-progress': { name: 'In Progress', items: [] },
    done: { name: 'Done', items: [] }
  });

  useEffect(() => {
    loadTasks();
  }, [userId]);

  const loadTasks = async () => {
    try {
      const tasks = await getTasks();
      const userTasks = tasks.filter(t => t.assignedTo === userId);

      const newColumns = {
        todo: { name: 'To Do', items: [] },
        'in-progress': { name: 'In Progress', items: [] },
        done: { name: 'Done', items: [] }
      };

      userTasks.forEach(task => {
        if (newColumns[task.status]) {
          newColumns[task.status].items.push(task);
        }
      });

      setColumns(newColumns);
    } catch (error) {
      console.error('Error loading tasks:', error);
    }
  };

  const onDragEnd = async (result) => {
    if (!result.destination) return;

    const { source, destination } = result;

    if (source.droppableId !== destination.droppableId) {
      const sourceColumn = columns[source.droppableId];
      const destColumn = columns[destination.droppableId];
      const sourceItems = [...sourceColumn.items];
      const destItems = [...destColumn.items];
      const [removed] = sourceItems.splice(source.index, 1);
      destItems.splice(destination.index, 0, removed);

      setColumns({
        ...columns,
        [source.droppableId]: {
          ...sourceColumn,
          items: sourceItems
        },
        [destination.droppableId]: {
          ...destColumn,
          items: destItems
        }
      });

      // Update task status in backend
      await updateTask(removed._id, { status: destination.droppableId });
    }
  };

  return (
    <DragDropContext onDragEnd={onDragEnd}>
      <div className="kanban-board">
        {Object.entries(columns).map(([columnId, column]) => (
          <div key={columnId} className="kanban-column">
            <h2>{column.name}</h2>
            <Droppable droppableId={columnId}>
              {(provided) => (
                <div
                  {...provided.droppableProps}
                  ref={provided.innerRef}
                  className="task-list"
                >
                  {column.items.map((task, index) => (
                    <Draggable
                      key={task._id}
                      draggableId={task._id}
                      index={index}
                    >
                      {(provided) => (
                        <div
                          ref={provided.innerRef}
                          {...provided.draggableProps}
                          {...provided.dragHandleProps}
                          className={`task-card priority-${task.priority}`}
                        >
                          <h3>{task.title}</h3>
                          <p>{task.description}</p>
                          <div className="task-meta">
                            <span className="priority">{task.priority}</span>
                            {task.dueDate && (
                              <span className="due-date">
                                Due: {new Date(task.dueDate).toLocaleDateString()}
                              </span>
                            )}
                          </div>
                        </div>
                      )}
                    </Draggable>
                  ))}
                  {provided.placeholder}
                </div>
              )}
            </Droppable>
          </div>
        ))}
      </div>
    </DragDropContext>
  );
};

export default KanbanBoard;
```

## Common Patterns

### Protected Routes with Role-Based Access

```javascript
// frontend/src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import ProtectedRoute from './components/ProtectedRoute';
import AdminDashboard from './components/AdminDashboard';
import UserDashboard from './components/UserDashboard';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route
          path="/admin"
          element={
            <ProtectedRoute requiredRole="admin">
              <AdminDashboard />
            </ProtectedRoute>
          }
        />
        <Route
          path="/dashboard"
          element={
            <ProtectedRoute>
              <UserDashboard />
            </ProtectedRoute>
          }
        />
      </Routes>
    </BrowserRouter>
  );
}
```

### Real-time Burnout Monitoring

```javascript
// backend/services/burnoutMonitor.js
const User = require('../models/User');
const axios = require('axios');

async function checkUserBurnout(userId) {
  const user = await User.findById(userId).populate('tasksAssigned');
  
  const completedTasks = user.tasksAssigned.filter(t => t.status === 'done').length;
  const totalTasks = user.tasksAssigned.length;
  const missedDeadlines = user.tasksAssigned.filter(t => {
    return t.dueDate && new Date(t.dueDate) < new Date() && t.status !== 'done';
  }).length;

  const behaviorData = {
    user_id: userId,
    login_frequency: user.loginFrequency || 10,
    task_completion_rate: totalTasks > 0 ? completedTasks / totalTasks : 1,
    average_task_time: user.avgTaskTime || 120,
    missed_deadlines: missedDeadlines,
    workload_score: user.workloadScore || 50,
    overtime_hours: user.overtimeHours || 0
  };

  const response = await axios.post(
    `${process.env.ML_SERVICE_URL}/api/ml/burnout-detection`,
    behaviorData
  );

  return response.data;
}

module.exports = { checkUserBurn
