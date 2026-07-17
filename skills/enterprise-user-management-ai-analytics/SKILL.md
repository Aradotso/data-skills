---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement user task tracking with AI insights"
  - "create admin dashboard with ML predictions"
  - "build ticket classification system"
  - "add burnout detection to user management"
  - "configure AI-powered user analytics"
  - "deploy enterprise management system with ML"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System, a full-stack application that combines user/task management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Admin analytics and user performance insights

The system consists of three main components:
1. **Frontend** (React.js) - User and admin interfaces
2. **Backend** (Node.js) - REST APIs and business logic
3. **ML Service** (FastAPI + scikit-learn) - AI/ML predictions

## Installation & Setup

### Prerequisites

```bash
# Required
node --version  # v14+
python --version  # 3.8+
mongod --version  # MongoDB 4.4+
```

### Complete Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# 1. Setup Backend
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_management
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start

# 2. Setup ML Service
cd ../ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
DATABASE_URL=mongodb://localhost:27017/enterprise_management
MODEL_PATH=./models
LOG_LEVEL=info
EOF

uvicorn main:app --reload --port 8000

# 3. Setup Frontend
cd ../frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

npm start
```

## Backend API Overview

### Authentication Endpoints

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const user = await User.create({
      username,
      email,
      password, // Should be hashed in User model pre-save hook
      role: role || 'user'
    });
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ success: true, token, user });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email }).select('+password');
    if (!user || !(await user.matchPassword(password))) {
      return res.status(401).json({ success: false, error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ success: true, token, user });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

### Task Management Endpoints

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { protect, authorize } = require('../middleware/auth');

// Get all tasks (user sees only their tasks, admin sees all)
router.get('/', protect, async (req, res) => {
  try {
    const query = req.user.role === 'admin' ? {} : { assignedTo: req.user.id };
    const tasks = await Task.find(query)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username');
    
    res.json({ success: true, data: tasks });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Create task (admin only)
router.post('/', protect, authorize('admin'), async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body; // 'todo', 'in-progress', 'done'
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ success: false, error: 'Task not found' });
    }
    
    // Users can only update their own tasks
    if (req.user.role !== 'admin' && task.assignedTo.toString() !== req.user.id) {
      return res.status(403).json({ success: false, error: 'Not authorized' });
    }
    
    task.status = status;
    task.timeTracking.push({
      status,
      timestamp: new Date(),
      duration: req.body.duration || 0
    });
    
    await task.save();
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

### Ticket Management with AI Integration

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { protect } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/api/classify-ticket`,
      {
        title,
        description,
        priority
      }
    );
    
    const ticket = await Ticket.create({
      title,
      description,
      priority,
      category: mlResponse.data.category,
      aiConfidence: mlResponse.data.confidence,
      assignedTeam: mlResponse.data.suggested_team,
      createdBy: req.user.id,
      status: 'open'
    });
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Get user risk assessment
router.get('/risk-assessment/:userId', protect, async (req, res) => {
  try {
    const mlResponse = await axios.get(
      `${process.env.ML_SERVICE_URL}/api/risk-assessment/${req.params.userId}`
    );
    
    res.json({ success: true, data: mlResponse.data });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

## ML Service API

### FastAPI Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, List
import os
from dotenv import load_dotenv

from models.ticket_classifier import TicketClassifier
from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector
from models.anomaly_detector import AnomalyDetector

load_dotenv()

app = FastAPI(title="Enterprise Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize ML models
ticket_classifier = TicketClassifier()
risk_predictor = RiskPredictor()
burnout_detector = BurnoutDetector()
anomaly_detector = AnomalyDetector()

class TicketData(BaseModel):
    title: str
    description: str
    priority: str

class UserBehavior(BaseModel):
    user_id: str
    login_times: List[int]
    failed_logins: int
    data_access_patterns: List[str]
    unusual_activities: int

class WorkloadData(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_task_duration: float
    overtime_hours: float
    missed_deadlines: int

@app.get("/")
async def root():
    return {"message": "Enterprise Management ML Service", "status": "active"}

@app.post("/api/classify-ticket")
async def classify_ticket(ticket: TicketData):
    """Classify support ticket and suggest team assignment"""
    try:
        result = ticket_classifier.classify(
            title=ticket.title,
            description=ticket.description,
            priority=ticket.priority
        )
        
        return {
            "category": result["category"],
            "confidence": result["confidence"],
            "suggested_team": result["team"],
            "estimated_resolution_time": result["eta_hours"]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/detect-anomaly")
async def detect_anomaly(behavior: UserBehavior):
    """Detect anomalous user behavior for security"""
    try:
        result = anomaly_detector.predict(
            user_id=behavior.user_id,
            login_times=behavior.login_times,
            failed_logins=behavior.failed_logins,
            access_patterns=behavior.data_access_patterns,
            unusual_activities=behavior.unusual_activities
        )
        
        return {
            "is_anomalous": result["is_anomaly"],
            "risk_score": result["score"],
            "anomaly_type": result["type"],
            "recommended_action": result["action"]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/detect-burnout")
async def detect_burnout(workload: WorkloadData):
    """Analyze workload to detect employee burnout risk"""
    try:
        result = burnout_detector.analyze(
            user_id=workload.user_id,
            tasks_assigned=workload.tasks_assigned,
            tasks_completed=workload.tasks_completed,
            avg_duration=workload.avg_task_duration,
            overtime=workload.overtime_hours,
            missed_deadlines=workload.missed_deadlines
        )
        
        return {
            "burnout_risk": result["risk_level"],  # low, medium, high
            "burnout_score": result["score"],
            "factors": result["contributing_factors"],
            "recommendations": result["suggestions"]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/risk-assessment/{user_id}")
async def get_risk_assessment(user_id: str):
    """Get comprehensive risk assessment for user"""
    try:
        result = risk_predictor.assess_risk(user_id)
        
        return {
            "user_id": user_id,
            "overall_risk": result["risk_level"],
            "risk_factors": result["factors"],
            "confidence": result["confidence"],
            "last_updated": result["timestamp"]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Ticket Classifier Implementation

```python
# ml-service/models/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
import pickle
import os

class TicketClassifier:
    def __init__(self):
        self.model_path = os.getenv('MODEL_PATH', './models')
        self.pipeline = self._load_or_create_model()
        
        # Category to team mapping
        self.team_mapping = {
            'technical': 'Engineering',
            'billing': 'Finance',
            'account': 'Customer Success',
            'feature_request': 'Product',
            'bug': 'Engineering',
            'general': 'Support'
        }
        
        # Resolution time estimates (hours)
        self.eta_mapping = {
            'low': 24,
            'medium': 12,
            'high': 4,
            'critical': 1
        }
    
    def _load_or_create_model(self):
        """Load existing model or create new one"""
        model_file = f"{self.model_path}/ticket_classifier.pkl"
        
        if os.path.exists(model_file):
            with open(model_file, 'rb') as f:
                return pickle.load(f)
        
        # Create new pipeline
        pipeline = Pipeline([
            ('tfidf', TfidfVectorizer(max_features=1000)),
            ('classifier', MultinomialNB())
        ])
        
        return pipeline
    
    def classify(self, title: str, description: str, priority: str):
        """Classify ticket and return category with metadata"""
        text = f"{title} {description}"
        
        # If model is trained
        if hasattr(self.pipeline.named_steps['classifier'], 'classes_'):
            prediction = self.pipeline.predict([text])[0]
            confidence = max(self.pipeline.predict_proba([text])[0])
        else:
            # Simple rule-based fallback
            prediction = self._rule_based_classification(text)
            confidence = 0.75
        
        return {
            'category': prediction,
            'confidence': float(confidence),
            'team': self.team_mapping.get(prediction, 'Support'),
            'eta_hours': self.eta_mapping.get(priority.lower(), 24)
        }
    
    def _rule_based_classification(self, text: str):
        """Fallback rule-based classification"""
        text_lower = text.lower()
        
        if any(word in text_lower for word in ['bug', 'error', 'crash', 'not working']):
            return 'bug'
        elif any(word in text_lower for word in ['billing', 'payment', 'invoice', 'charge']):
            return 'billing'
        elif any(word in text_lower for word in ['feature', 'request', 'add', 'need']):
            return 'feature_request'
        elif any(word in text_lower for word in ['account', 'login', 'password', 'access']):
            return 'account'
        elif any(word in text_lower for word in ['api', 'integration', 'code', 'technical']):
            return 'technical'
        else:
            return 'general'
```

### Burnout Detector Implementation

```python
# ml-service/models/burnout_detector.py
import numpy as np
from sklearn.preprocessing import StandardScaler

class BurnoutDetector:
    def __init__(self):
        self.scaler = StandardScaler()
        
        # Thresholds for risk assessment
        self.thresholds = {
            'high': 0.7,
            'medium': 0.4,
            'low': 0.2
        }
    
    def analyze(self, user_id: str, tasks_assigned: int, tasks_completed: int,
                avg_duration: float, overtime: float, missed_deadlines: int):
        """Analyze workload patterns to detect burnout risk"""
        
        # Calculate metrics
        completion_rate = tasks_completed / max(tasks_assigned, 1)
        workload_ratio = tasks_assigned / 10  # Normalized to 10 tasks baseline
        overtime_factor = overtime / 40  # Weekly hours baseline
        deadline_miss_rate = missed_deadlines / max(tasks_assigned, 1)
        
        # Calculate burnout score (0-1)
        burnout_score = self._calculate_score(
            completion_rate,
            workload_ratio,
            overtime_factor,
            deadline_miss_rate
        )
        
        # Determine risk level
        if burnout_score >= self.thresholds['high']:
            risk_level = 'high'
        elif burnout_score >= self.thresholds['medium']:
            risk_level = 'medium'
        else:
            risk_level = 'low'
        
        # Identify contributing factors
        factors = self._identify_factors(
            completion_rate, workload_ratio, overtime_factor, deadline_miss_rate
        )
        
        # Generate recommendations
        suggestions = self._generate_recommendations(risk_level, factors)
        
        return {
            'risk_level': risk_level,
            'score': float(burnout_score),
            'contributing_factors': factors,
            'suggestions': suggestions
        }
    
    def _calculate_score(self, completion_rate, workload_ratio, 
                        overtime_factor, deadline_miss_rate):
        """Calculate weighted burnout score"""
        
        # Inverse completion rate (lower completion = higher burnout)
        completion_penalty = 1 - completion_rate
        
        # Combine factors with weights
        score = (
            completion_penalty * 0.25 +
            min(workload_ratio, 1) * 0.30 +
            min(overtime_factor, 1) * 0.25 +
            min(deadline_miss_rate, 1) * 0.20
        )
        
        return np.clip(score, 0, 1)
    
    def _identify_factors(self, completion_rate, workload_ratio,
                         overtime_factor, deadline_miss_rate):
        """Identify main contributing factors"""
        factors = []
        
        if completion_rate < 0.7:
            factors.append('low_task_completion')
        if workload_ratio > 1.5:
            factors.append('excessive_workload')
        if overtime_factor > 0.25:
            factors.append('high_overtime')
        if deadline_miss_rate > 0.3:
            factors.append('frequent_deadline_misses')
        
        return factors if factors else ['normal_workload']
    
    def _generate_recommendations(self, risk_level, factors):
        """Generate actionable recommendations"""
        recommendations = []
        
        if risk_level == 'high':
            recommendations.append('Immediate workload review required')
            recommendations.append('Schedule one-on-one with manager')
        
        if 'excessive_workload' in factors:
            recommendations.append('Redistribute tasks to team members')
        
        if 'high_overtime' in factors:
            recommendations.append('Enforce work-life balance policies')
        
        if 'frequent_deadline_misses' in factors:
            recommendations.append('Review task complexity and deadlines')
        
        if not recommendations:
            recommendations.append('Maintain current workload balance')
        
        return recommendations
```

## Frontend Integration

### API Service Client

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

// Create axios instance with auth token
const api = axios.create({
  baseURL: API_URL
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authAPI = {
  login: (credentials) => api.post('/api/auth/login', credentials),
  register: (userData) => api.post('/api/auth/register', userData),
  getCurrentUser: () => api.get('/api/auth/me')
};

export const taskAPI = {
  getTasks: () => api.get('/api/tasks'),
  createTask: (taskData) => api.post('/api/tasks', taskData),
  updateTaskStatus: (taskId, status, duration) => 
    api.patch(`/api/tasks/${taskId}/status`, { status, duration }),
  deleteTask: (taskId) => api.delete(`/api/tasks/${taskId}`)
};

export const ticketAPI = {
  createTicket: (ticketData) => api.post('/api/tickets', ticketData),
  getTickets: () => api.get('/api/tickets'),
  updateTicket: (ticketId, updates) => api.patch(`/api/tickets/${ticketId}`, updates)
};

export const mlAPI = {
  getRiskAssessment: (userId) => 
    axios.get(`${ML_API_URL}/api/risk-assessment/${userId}`),
  detectBurnout: (workloadData) => 
    axios.post(`${ML_API_URL}/api/detect-burnout`, workloadData),
  detectAnomaly: (behaviorData) => 
    axios.post(`${ML_API_URL}/api/detect-anomaly`, behaviorData)
};

export default api;
```

### Task Board Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';
import './TaskBoard.css';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    try {
      const response = await taskAPI.getTasks();
      const tasksByStatus = {
        todo: response.data.data.filter(t => t.status === 'todo'),
        inProgress: response.data.data.filter(t => t.status === 'in-progress'),
        done: response.data.data.filter(t => t.status === 'done')
      };
      setTasks(tasksByStatus);
      setLoading(false);
    } catch (error) {
      console.error('Failed to load tasks:', error);
      setLoading(false);
    }
  };

  const handleStatusChange = async (taskId, newStatus, duration = 0) => {
    try {
      await taskAPI.updateTaskStatus(taskId, newStatus, duration);
      loadTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        <span className="due-date">{new Date(task.dueDate).toLocaleDateString()}</span>
      </div>
      <div className="task-actions">
        {task.status !== 'done' && (
          <button onClick={() => handleStatusChange(
            task._id, 
            task.status === 'todo' ? 'in-progress' : 'done'
          )}>
            {task.status === 'todo' ? 'Start' : 'Complete'}
          </button>
        )}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="task-board">
      <div className="column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default TaskBoard;
```

### Admin Dashboard with AI Insights

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { mlAPI, taskAPI } from '../services/api';
import './AdminDashboard.css';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [burnoutRisks, setBurnoutRisks] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      // Load tasks and calculate burnout risks
      const tasksResponse = await taskAPI.getTasks();
      const tasks = tasksResponse.data.data;
      
      // Group by user and analyze
      const userWorkloads = calculateUserWorkloads(tasks);
      
      // Get burnout predictions for each user
      const burnoutPromises = Object.keys(userWorkloads).map(userId =>
        mlAPI.detectBurnout(userWorkloads[userId])
          .then(res => ({ userId, ...res.data }))
          .catch(() => null)
      );
      
      const burnoutResults = (await Promise.all(burnoutPromises))
        .filter(r => r && r.burnout_risk !== 'low');
      
      setBurnoutRisks(burnoutResults);
      setAnalytics({
        totalTasks: tasks.length,
        completedTasks: tasks.filter(t => t.status === 'done').length,
        inProgressTasks: tasks.filter(t => t.status === 'in-progress').length,
        highRiskUsers: burnoutResults.filter(r => r.burnout_risk === 'high').length
      });
      setLoading(false);
    } catch (error) {
      console.error('Failed to load dashboard data:', error);
      setLoading(false);
    }
  };

  const calculateUserWorkloads = (tasks) => {
    const workloads = {};
    
    tasks.forEach(task => {
      const userId = task.assignedTo._id;
      if (!workloads[userId]) {
        workloads[userId] = {
          user_id: userId,
          tasks_assigned: 0,
          tasks_completed: 0,
          avg_task_duration: 0,
          overtime_hours: 0,
          missed_deadlines: 0
        };
      }
      
      workloads[userId].tasks_assigned++;
      if (task.status === 'done') workloads[userId].tasks_completed++;
      if (new Date(task.dueDate) < new Date() && task.status !== 'done') {
        workloads[userId].missed_deadlines++;
      }
    });
    
    return workloads;
  };

  if (loading) return <div>Loading dashboard...</div>;

  return (
    <div className="admin-dashboard">
      <h2>Admin Dashboard</h2>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Tasks</h3>
          <p className="stat-value">{analytics.totalTasks}</p>
        </div>
        <div className="stat-card">
          <h3>Completed</h3>
          <p className="stat-value">{analytics.completedTasks}</p>
        </div>
        <div className="stat-card">
          <h3>In Progress</h3>
          <p className="stat-value">{analytics.inProgressTasks}</p>
        </div>
        <div className="stat-card alert">
          <h3>High Risk Users</h3>
          <p className="stat-value">{analytics.highRiskUsers}</p>
        </div>
      </div>

      <div className="burnout-alerts">
        <h3>Burnout Risk Alerts</h3>
        {burnoutRisks.length === 0 ? (
          <p>No high-risk users detected</p>
        ) : (
          <table>
            <thead>
              <tr>
                <th>User ID</th>
                <th>Risk Level</th>
                <th>Score</th>
                <th>Factors</th>
                <th>Recommendations</th>
              </tr>
            </thead>
            <tbody>
              {burnoutRisks.map(risk => (
                <tr key={risk.userId} className={`risk-${risk.burnout_risk}`}>
                  <td>{risk.userId}</td>
                  <td>{risk.burnout_risk}</td>
                  <td>{(risk.burnout_score * 100).toFixed(1)}%</td>
                  <td>{risk.factors.join(', ')}</td>
                  <td>
                    <ul>
                      {risk.recommendations.map((rec, i) => (
                        <li key={i}>{rec}</li>
                      ))}
                    </ul>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        )}
      </div>
    </div>
  );
};

export
