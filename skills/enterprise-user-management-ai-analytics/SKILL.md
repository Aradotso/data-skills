---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "configure AI analytics for user management"
  - "implement task tracking with kanban board"
  - "create support ticket system with AI classification"
  - "add burnout detection and risk prediction"
  - "build admin dashboard for user management"
  - "integrate ML service for anomaly detection"
  - "deploy user management system with AI features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System, a full-stack application that combines user/task management with AI-powered analytics including risk detection, burnout analysis, and predictive insights.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Admin analytics and user performance insights

**Architecture**: React frontend + Node.js/Express backend + FastAPI ML service + MongoDB database

## Installation

### Prerequisites
```bash
# Required software
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
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
MONGODB_URI=mongodb://localhost:27017/user-management
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
# Runs on http://localhost:5000
```

### ML Service Setup
```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/user-management
MODEL_PATH=./models
LOG_LEVEL=info
EOF

uvicorn main:app --reload --port 8000
# Runs on http://localhost:8000
```

### Frontend Setup
```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
EOF

npm start
# Runs on http://localhost:3000
```

## Backend API Structure

### User Authentication
```javascript
// backend/controllers/authController.js
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// Register new user
exports.register = async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const hashedPassword = await bcrypt.hash(password, 10);
    
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
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
};

// Login user
exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ 
        success: false, 
        message: 'Invalid credentials' 
      });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ 
        success: false, 
        message: 'Invalid credentials' 
      });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
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
};
```

### Task Management
```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

// Create task
exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      status: 'todo',
      dueDate,
      createdBy: req.user.userId
    });
    
    await task.save();
    
    res.status(201).json({
      success: true,
      task
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Update task status (Kanban)
exports.updateTaskStatus = async (req, res) => {
  try {
    const { taskId } = req.params;
    const { status } = req.body;
    
    const task = await Task.findById(taskId);
    if (!task) {
      return res.status(404).json({ 
        success: false, 
        message: 'Task not found' 
      });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    
    res.json({
      success: true,
      task
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Get user tasks
exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ 
      assignedTo: req.user.userId 
    }).populate('createdBy', 'username email');
    
    res.json({
      success: true,
      tasks
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

### Support Ticket System
```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

// Create ticket with AI classification
exports.createTicket = async (req, res) => {
  try {
    const { subject, description, priority } = req.body;
    
    // Get AI classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      {
        subject,
        description
      }
    );
    
    const { category, suggestedPriority, autoAssign } = mlResponse.data;
    
    const ticket = new Ticket({
      subject,
      description,
      priority: priority || suggestedPriority,
      category,
      status: 'open',
      createdBy: req.user.userId,
      assignedTo: autoAssign || null
    });
    
    await ticket.save();
    
    res.status(201).json({
      success: true,
      ticket,
      aiInsights: {
        category,
        suggestedPriority
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Get tickets with filters
exports.getTickets = async (req, res) => {
  try {
    const { status, category, priority } = req.query;
    
    const filter = {};
    if (status) filter.status = status;
    if (category) filter.category = category;
    if (priority) filter.priority = priority;
    
    // Non-admin users see only their tickets
    if (req.user.role !== 'admin') {
      filter.createdBy = req.user.userId;
    }
    
    const tickets = await Ticket.find(filter)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json({
      success: true,
      tickets
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

## ML Service (FastAPI)

### Main Application Setup
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="User Management AI Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketClassification(BaseModel):
    subject: str
    description: str

class RiskPrediction(BaseModel):
    userId: str
    recentActivity: dict
    taskMetrics: dict

class BurnoutAnalysis(BaseModel):
    userId: str
    workload: int
    overtimeHours: float
    taskCompletionRate: float
    daysWithoutBreak: int

@app.get("/")
def root():
    return {"status": "AI Service Running", "version": "1.0"}

@app.post("/classify-ticket")
def classify_ticket(data: TicketClassification):
    """Classify support ticket using NLP"""
    try:
        # Simple keyword-based classification
        text = (data.subject + " " + data.description).lower()
        
        categories = {
            'technical': ['bug', 'error', 'crash', 'issue', 'problem'],
            'account': ['password', 'login', 'access', 'account'],
            'feature': ['request', 'feature', 'enhancement', 'add'],
            'performance': ['slow', 'performance', 'speed', 'lag']
        }
        
        scores = {}
        for category, keywords in categories.items():
            scores[category] = sum(1 for kw in keywords if kw in text)
        
        category = max(scores, key=scores.get) if max(scores.values()) > 0 else 'general'
        
        # Determine priority based on keywords
        urgent_keywords = ['urgent', 'critical', 'down', 'broken', 'immediately']
        priority = 'high' if any(kw in text for kw in urgent_keywords) else 'medium'
        
        return {
            "category": category,
            "suggestedPriority": priority,
            "confidence": max(scores.values()) / len(text.split()) if text else 0,
            "autoAssign": None  # Could integrate assignment logic
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
def predict_risk(data: RiskPrediction):
    """Predict user risk based on behavior patterns"""
    try:
        activity = data.recentActivity
        metrics = data.taskMetrics
        
        # Risk scoring algorithm
        risk_score = 0
        risk_factors = []
        
        # Failed login attempts
        failed_logins = activity.get('failedLogins', 0)
        if failed_logins > 3:
            risk_score += 20
            risk_factors.append(f"High failed login attempts: {failed_logins}")
        
        # Unusual access times
        unusual_access = activity.get('unusualAccessTimes', 0)
        if unusual_access > 5:
            risk_score += 15
            risk_factors.append("Unusual access patterns detected")
        
        # Task completion rate
        completion_rate = metrics.get('completionRate', 100)
        if completion_rate < 50:
            risk_score += 10
            risk_factors.append(f"Low task completion: {completion_rate}%")
        
        # Overdue tasks
        overdue = metrics.get('overdueTasks', 0)
        if overdue > 3:
            risk_score += 15
            risk_factors.append(f"Multiple overdue tasks: {overdue}")
        
        # Determine risk level
        if risk_score >= 50:
            risk_level = "high"
        elif risk_score >= 25:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return {
            "riskLevel": risk_level,
            "riskScore": risk_score,
            "riskFactors": risk_factors,
            "recommendations": get_recommendations(risk_level, risk_factors)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
def detect_burnout(data: BurnoutAnalysis):
    """Detect employee burnout using workload metrics"""
    try:
        burnout_score = 0
        indicators = []
        
        # Workload analysis
        if data.workload > 80:
            burnout_score += 25
            indicators.append("Excessive workload")
        
        # Overtime analysis
        if data.overtimeHours > 10:
            burnout_score += 20
            indicators.append(f"High overtime: {data.overtimeHours} hrs/week")
        
        # Task completion rate
        if data.taskCompletionRate < 70:
            burnout_score += 15
            indicators.append(f"Declining performance: {data.taskCompletionRate}%")
        
        # Days without break
        if data.daysWithoutBreak > 14:
            burnout_score += 30
            indicators.append(f"No breaks for {data.daysWithoutBreak} days")
        
        # Calculate burnout risk
        if burnout_score >= 60:
            burnout_risk = "critical"
            action = "Immediate intervention required"
        elif burnout_score >= 40:
            burnout_risk = "high"
            action = "Review workload distribution"
        elif burnout_score >= 20:
            burnout_risk = "moderate"
            action = "Monitor situation closely"
        else:
            burnout_risk = "low"
            action = "Continue regular monitoring"
        
        return {
            "burnoutRisk": burnout_risk,
            "burnoutScore": burnout_score,
            "indicators": indicators,
            "recommendedAction": action,
            "suggestedBreakDays": max(0, data.daysWithoutBreak - 7)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-project-delay")
def predict_project_delay(projectData: dict):
    """Predict likelihood of project delay"""
    try:
        total_tasks = projectData.get('totalTasks', 0)
        completed_tasks = projectData.get('completedTasks', 0)
        days_elapsed = projectData.get('daysElapsed', 0)
        total_days = projectData.get('totalDays', 0)
        
        if total_tasks == 0 or total_days == 0:
            raise HTTPException(status_code=400, detail="Invalid project data")
        
        # Calculate progress metrics
        task_completion_rate = (completed_tasks / total_tasks) * 100
        time_elapsed_rate = (days_elapsed / total_days) * 100
        
        # Predict delay
        delay_indicator = time_elapsed_rate - task_completion_rate
        
        if delay_indicator > 20:
            delay_risk = "high"
            predicted_delay_days = int(total_days * (delay_indicator / 100))
        elif delay_indicator > 10:
            delay_risk = "medium"
            predicted_delay_days = int(total_days * (delay_indicator / 200))
        else:
            delay_risk = "low"
            predicted_delay_days = 0
        
        return {
            "delayRisk": delay_risk,
            "taskCompletionRate": round(task_completion_rate, 2),
            "timeElapsedRate": round(time_elapsed_rate, 2),
            "predictedDelayDays": predicted_delay_days,
            "recommendations": get_project_recommendations(delay_risk)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_recommendations(risk_level, risk_factors):
    """Get security recommendations based on risk"""
    recommendations = []
    
    if "failed login" in str(risk_factors).lower():
        recommendations.append("Review login attempts and consider password reset")
    if "unusual access" in str(risk_factors).lower():
        recommendations.append("Verify user identity and recent activities")
    if "overdue" in str(risk_factors).lower():
        recommendations.append("Reassign tasks or extend deadlines")
    
    if risk_level == "high":
        recommendations.append("Consider temporary account restriction")
    
    return recommendations

def get_project_recommendations(delay_risk):
    """Get project management recommendations"""
    if delay_risk == "high":
        return [
            "Increase team resources",
            "Reprioritize tasks",
            "Review project scope"
        ]
    elif delay_risk == "medium":
        return [
            "Monitor progress daily",
            "Identify bottlenecks",
            "Consider overtime if necessary"
        ]
    else:
        return ["Continue current pace", "Maintain quality standards"]
```

## Frontend Integration

### API Service Setup
```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';
const ML_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

// Axios instance with auth
const api = axios.create({
  baseURL: API_URL
});

// Add auth token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

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
  }
};

// Task services
export const taskService = {
  getTasks: async () => {
    const response = await api.get('/tasks');
    return response.data.tasks;
  },
  
  createTask: async (taskData) => {
    const response = await api.post('/tasks', taskData);
    return response.data.task;
  },
  
  updateTaskStatus: async (taskId, status) => {
    const response = await api.patch(`/tasks/${taskId}/status`, { status });
    return response.data.task;
  },
  
  deleteTask: async (taskId) => {
    const response = await api.delete(`/tasks/${taskId}`);
    return response.data;
  }
};

// Ticket services
export const ticketService = {
  getTickets: async (filters = {}) => {
    const response = await api.get('/tickets', { params: filters });
    return response.data.tickets;
  },
  
  createTicket: async (ticketData) => {
    const response = await api.post('/tickets', ticketData);
    return response.data;
  },
  
  updateTicket: async (ticketId, updates) => {
    const response = await api.patch(`/tickets/${ticketId}`, updates);
    return response.data.ticket;
  }
};

// AI/ML services
export const mlService = {
  getRiskPrediction: async (userId, activityData) => {
    const response = await axios.post(`${ML_URL}/predict-risk`, {
      userId,
      recentActivity: activityData.recentActivity,
      taskMetrics: activityData.taskMetrics
    });
    return response.data;
  },
  
  getBurnoutAnalysis: async (userId, metrics) => {
    const response = await axios.post(`${ML_URL}/detect-burnout`, {
      userId,
      ...metrics
    });
    return response.data;
  },
  
  getProjectDelayPrediction: async (projectData) => {
    const response = await axios.post(`${ML_URL}/predict-project-delay`, projectData);
    return response.data;
  }
};

export default api;
```

### Kanban Board Component
```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    try {
      const allTasks = await taskService.getTasks();
      
      const grouped = {
        todo: allTasks.filter(t => t.status === 'todo'),
        inProgress: allTasks.filter(t => t.status === 'inProgress'),
        done: allTasks.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to load tasks:', error);
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
      await taskService.updateTaskStatus(taskId, newStatus);
      await loadTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const renderColumn = (title, status, taskList) => (
    <div 
      className="kanban-column"
      onDrop={(e) => handleDrop(e, status)}
      onDragOver={handleDragOver}
    >
      <h3>{title} ({taskList.length})</h3>
      <div className="task-list">
        {taskList.map(task => (
          <div
            key={task._id}
            className={`task-card priority-${task.priority}`}
            draggable
            onDragStart={(e) => handleDragStart(e, task._id)}
          >
            <h4>{task.title}</h4>
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
        ))}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {renderColumn('To Do', 'todo', tasks.todo)}
      {renderColumn('In Progress', 'inProgress', tasks.inProgress)}
      {renderColumn('Done', 'done', tasks.done)}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard
```javascript
// frontend/src/components/AIAnalyticsDashboard.js
import React, { useState, useEffect } from 'react';
import { mlService } from '../services/api';
import './AIAnalyticsDashboard.css';

const AIAnalyticsDashboard = ({ userId, userMetrics }) => {
  const [analytics, setAnalytics] = useState({
    riskPrediction: null,
    burnoutAnalysis: null,
    loading: true
  });

  useEffect(() => {
    loadAnalytics();
  }, [userId]);

  const loadAnalytics = async () => {
    try {
      const [risk, burnout] = await Promise.all([
        mlService.getRiskPrediction(userId, {
          recentActivity: {
            failedLogins: userMetrics.failedLogins || 0,
            unusualAccessTimes: userMetrics.unusualAccessTimes || 0
          },
          taskMetrics: {
            completionRate: userMetrics.taskCompletionRate || 100,
            overdueTasks: userMetrics.overdueTasks || 0
          }
        }),
        mlService.getBurnoutAnalysis(userId, {
          workload: userMetrics.workload || 0,
          overtimeHours: userMetrics.overtimeHours || 0,
          taskCompletionRate: userMetrics.taskCompletionRate || 100,
          daysWithoutBreak: userMetrics.daysWithoutBreak || 0
        })
      ]);

      setAnalytics({
        riskPrediction: risk,
        burnoutAnalysis: burnout,
        loading: false
      });
    } catch (error) {
      console.error('Failed to load AI analytics:', error);
      setAnalytics(prev => ({ ...prev, loading: false }));
    }
  };

  if (analytics.loading) return <div>Loading AI insights...</div>;

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI-Powered Insights</h2>
      
      {/* Risk Prediction */}
      {analytics.riskPrediction && (
        <div className={`analytics-card risk-${analytics.riskPrediction.riskLevel}`}>
          <h3>Risk Assessment</h3>
          <div className="risk-level">
            Risk Level: <strong>{analytics.riskPrediction.riskLevel.toUpperCase()}</strong>
          </div>
          <div className="risk-score">
            Score: {analytics.riskPrediction.riskScore}/100
          </div>
          
          {analytics.riskPrediction.riskFactors.length > 0 && (
            <div className="risk-factors">
              <h4>Risk Factors:</h4>
              <ul>
                {analytics.riskPrediction.riskFactors.map((factor, idx) => (
                  <li key={idx}>{factor}</li>
                ))}
              </ul>
            </div>
          )}
          
          {analytics.riskPrediction.recommendations.length > 0 && (
            <div className="recommendations">
              <h4>Recommendations:</h4>
              <ul>
                {analytics.riskPrediction.recommendations.map((rec, idx) => (
                  <li key={idx}>{rec}</li>
                ))}
              </ul>
            </div>
          )}
        </div>
      )}
      
      {/* Burnout Analysis */}
      {analytics.burnoutAnalysis && (
        <div className={`analytics-card burnout-${analytics.burnoutAnalysis.burnoutRisk}`}>
          <h3>Burnout Detection</h3>
          <div className="burnout-level">
            Burnout Risk: <strong>{analytics.burnoutAnalysis.burnoutRisk.toUpperCase()}</strong>
          </div>
          <div className="burnout-score">
            Score: {analytics.burnoutAnalysis.burnoutScore}/100
          </div>
          
          {analytics.burnoutAnalysis.indicators.length > 0 && (
            <div className="burnout-indicators">
              <h4>Indicators:</h4>
              <ul>
                {analytics.burnoutAnalysis.indicators.map((indicator, idx) => (
                  <li key={idx}>{indicator}</li>
                ))}
              </ul>
            </div>
          )}
          
          <div className="recommended-action">
            <h4>Recommended Action:</h4>
            <p>{analytics.burnoutAnalysis.recommendedAction}</p>
          </div>
          
          {analytics.burnoutAnalysis.suggestedBreakDays > 0 && (
            <div className="break-suggestion">
              Suggested break: {analytics.burnoutAnalysis.suggestedBreakDays} days
            </div>
          )}
        </div>
      )}
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Database Models

### User Model
```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true
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
  profileImage: String,
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

module.exports = mongoose.model('User', userSchema);
```

### Task Model
```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    
