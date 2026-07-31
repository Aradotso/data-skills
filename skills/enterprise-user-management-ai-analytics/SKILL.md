```markdown
---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "configure AI analytics for user management"
  - "implement ticket classification with ML"
  - "add risk detection to user system"
  - "create user dashboard with AI insights"
  - "integrate anomaly detection for users"
  - "build task management with burnout analysis"
  - "deploy user management with predictive analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines traditional user/task management with AI-powered insights. The system provides:

- User authentication and role-based access control (RBAC)
- Task management with Kanban boards
- Support ticket system with AI classification
- AI analytics: risk prediction, anomaly detection, burnout analysis
- Admin dashboards with predictive insights
- Real-time notifications and audit logs

**Architecture**: React frontend + Node.js/Express backend + FastAPI ML service + MongoDB

## Installation

### Prerequisites

```bash
# Required
node >= 14.0.0
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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=\${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
CORS_ORIGINS=http://localhost:3000,http://localhost:5000
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Key API Endpoints

### Authentication

```javascript
// POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@company.com",
  "password": "securePassword123",
  "role": "user" // or "admin"
}

// POST /api/auth/login
{
  "email": "john@company.com",
  "password": "securePassword123"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "507f1f77bcf86cd799439011",
    "name": "John Doe",
    "email": "john@company.com",
    "role": "user"
  }
}
```

### User Management (Admin)

```javascript
// GET /api/users - List all users
// GET /api/users/:id - Get user details
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user

// Example: Update user
fetch('http://localhost:5000/api/users/507f1f77bcf86cd799439011', {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    name: "John Smith",
    role: "admin",
    status: "active"
  })
});
```

### Task Management

```javascript
// POST /api/tasks - Create task
{
  "title": "Implement user authentication",
  "description": "Add JWT-based auth system",
  "assignedTo": "507f1f77bcf86cd799439011",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01T00:00:00Z"
}

// PUT /api/tasks/:id/status
{
  "status": "in-progress" // or "done"
}

// POST /api/tasks/:id/time - Track time
{
  "timeSpent": 3600 // seconds
}
```

### Support Tickets

```javascript
// POST /api/tickets - Create ticket
{
  "subject": "Login issue",
  "description": "Cannot login with correct credentials",
  "priority": "high",
  "category": "technical" // auto-classified by AI
}

// GET /api/tickets - List tickets (filtered by user/admin)
// PUT /api/tickets/:id - Update ticket status
{
  "status": "in-progress",
  "assignedTo": "admin-user-id"
}
```

## ML Service API

### Ticket Classification

```python
# POST /classify-ticket
import requests

response = requests.post(
    'http://localhost:8000/classify-ticket',
    json={
        "subject": "Password reset not working",
        "description": "I clicked forgot password but didn't receive email"
    }
)

# Response
{
  "category": "technical",
  "priority": "high",
  "confidence": 0.87,
  "routing_suggestion": "IT Support Team"
}
```

### Risk Detection

```python
# POST /detect-risk
response = requests.post(
    'http://localhost:8000/detect-risk',
    json={
        "user_id": "507f1f77bcf86cd799439011",
        "login_attempts": 5,
        "failed_logins": 4,
        "unusual_access_time": True,
        "location_changed": True,
        "data_access_volume": 1500
    }
)

# Response
{
  "risk_score": 0.72,
  "risk_level": "high",
  "factors": ["multiple_failed_logins", "unusual_time", "location_change"],
  "recommendation": "Require additional verification"
}
```

### Burnout Detection

```python
# POST /detect-burnout
response = requests.post(
    'http://localhost:8000/detect-burnout',
    json={
        "user_id": "507f1f77bcf86cd799439011",
        "hours_worked_week": 65,
        "tasks_completed": 3,
        "tasks_overdue": 8,
        "avg_task_completion_time": 120, # hours
        "tickets_raised": 12
    }
)

# Response
{
  "burnout_score": 0.81,
  "burnout_level": "high",
  "indicators": ["overwork", "low_completion_rate", "high_stress"],
  "recommendation": "Redistribute workload, schedule check-in"
}
```

### Anomaly Detection

```python
# POST /detect-anomaly
response = requests.post(
    'http://localhost:8000/detect-anomaly',
    json={
        "user_id": "507f1f77bcf86cd799439011",
        "action": "data_export",
        "timestamp": "2026-04-15T03:30:00Z",
        "data_volume": 50000,
        "ip_address": "192.168.1.100",
        "device": "unknown"
    }
)

# Response
{
  "is_anomaly": True,
  "anomaly_score": 0.89,
  "reasons": ["unusual_time", "high_volume", "new_device"],
  "action_required": "Investigate immediately"
}
```

## Code Examples

### Frontend: User Dashboard Component

```javascript
// src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [aiInsights, setAiInsights] = useState(null);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
    fetchAIInsights();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks/my-tasks`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      // Group by status
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const fetchAIInsights = async () => {
    try {
      const userId = JSON.parse(localStorage.getItem('user')).id;
      const response = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/detect-burnout`,
        {
          user_id: userId,
          hours_worked_week: 45,
          tasks_completed: tasks.done.length,
          tasks_overdue: tasks.todo.filter(t => new Date(t.dueDate) < new Date()).length
        }
      );
      setAiInsights(response.data);
    } catch (error) {
      console.error('Failed to fetch AI insights:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="dashboard">
      {aiInsights && aiInsights.burnout_level === 'high' && (
        <div className="alert alert-warning">
          ⚠️ High burnout risk detected. {aiInsights.recommendation}
        </div>
      )}
      
      <div className="kanban-board">
        <div className="column">
          <h3>To Do ({tasks.todo.length})</h3>
          {tasks.todo.map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
        
        <div className="column">
          <h3>In Progress ({tasks.inProgress.length})</h3>
          {tasks.inProgress.map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
        
        <div className="column">
          <h3>Done ({tasks.done.length})</h3>
          {tasks.done.map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Backend: Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

// Create task
exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = await Task.create({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      createdBy: req.user.id,
      status: 'todo'
    });

    // Log in audit trail
    await logAuditEvent({
      userId: req.user.id,
      action: 'TASK_CREATED',
      resource: 'task',
      resourceId: task._id
    });

    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Update task status
exports.updateTaskStatus = async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    // Check authorization
    if (task.assignedTo.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Not authorized' });
    }

    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    await task.save();

    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Get my tasks
exports.getMyTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Track time spent
exports.trackTime = async (req, res) => {
  try {
    const { timeSpent } = req.body; // in seconds
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.timeSpent = (task.timeSpent || 0) + timeSpent;
    await task.save();

    res.json({ timeSpent: task.timeSpent });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

### ML Service: Ticket Classification Model

```python
# ml-service/models/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

class TicketClassifier:
    def __init__(self, model_path='./models'):
        self.model_path = model_path
        self.vectorizer = TfidfVectorizer(max_features=1000)
        self.classifier = RandomForestClassifier(n_estimators=100)
        self.categories = ['technical', 'billing', 'access', 'general']
        self.priorities = ['low', 'medium', 'high', 'critical']
        
        # Load pre-trained model if exists
        if os.path.exists(f'{model_path}/ticket_classifier.pkl'):
            self.load_model()
    
    def preprocess(self, subject, description):
        """Combine and preprocess ticket text"""
        text = f"{subject} {description}".lower()
        return text
    
    def classify(self, subject, description):
        """Classify ticket category and priority"""
        text = self.preprocess(subject, description)
        
        # Simple rule-based classification (can be enhanced with ML)
        category = self._predict_category(text)
        priority = self._predict_priority(text, category)
        confidence = self._calculate_confidence(text, category)
        
        return {
            'category': category,
            'priority': priority,
            'confidence': confidence,
            'routing_suggestion': self._get_routing(category)
        }
    
    def _predict_category(self, text):
        """Predict ticket category"""
        keywords = {
            'technical': ['error', 'bug', 'crash', 'login', 'password', 'system'],
            'billing': ['payment', 'invoice', 'charge', 'refund', 'subscription'],
            'access': ['permission', 'access', 'role', 'cannot view', 'locked'],
            'general': ['question', 'how to', 'information', 'request']
        }
        
        scores = {}
        for category, words in keywords.items():
            scores[category] = sum(1 for word in words if word in text)
        
        return max(scores, key=scores.get) if max(scores.values()) > 0 else 'general'
    
    def _predict_priority(self, text, category):
        """Predict ticket priority"""
        urgent_keywords = ['urgent', 'critical', 'emergency', 'down', 'broken', 'asap']
        high_keywords = ['important', 'soon', 'issue', 'problem', 'not working']
        
        if any(word in text for word in urgent_keywords):
            return 'critical'
        elif any(word in text for word in high_keywords):
            return 'high'
        elif category == 'technical':
            return 'medium'
        else:
            return 'low'
    
    def _calculate_confidence(self, text, category):
        """Calculate classification confidence"""
        # Simplified confidence calculation
        word_count = len(text.split())
        return min(0.5 + (word_count * 0.05), 0.95)
    
    def _get_routing(self, category):
        """Get routing suggestion based on category"""
        routing = {
            'technical': 'IT Support Team',
            'billing': 'Finance Department',
            'access': 'Security Team',
            'general': 'Customer Service'
        }
        return routing.get(category, 'General Support')
    
    def save_model(self):
        """Save trained model"""
        os.makedirs(self.model_path, exist_ok=True)
        joblib.dump(self.classifier, f'{self.model_path}/ticket_classifier.pkl')
        joblib.dump(self.vectorizer, f'{self.model_path}/ticket_vectorizer.pkl')
    
    def load_model(self):
        """Load pre-trained model"""
        self.classifier = joblib.load(f'{self.model_path}/ticket_classifier.pkl')
        self.vectorizer = joblib.load(f'{self.model_path}/ticket_vectorizer.pkl')
```

### ML Service: FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional
import os
from models.ticket_classifier import TicketClassifier
from models.risk_detector import RiskDetector
from models.burnout_analyzer import BurnoutAnalyzer

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=os.getenv("CORS_ORIGINS", "http://localhost:3000").split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize models
ticket_classifier = TicketClassifier()
risk_detector = RiskDetector()
burnout_analyzer = BurnoutAnalyzer()

# Request models
class TicketRequest(BaseModel):
    subject: str
    description: str

class RiskRequest(BaseModel):
    user_id: str
    login_attempts: int
    failed_logins: int
    unusual_access_time: bool
    location_changed: bool
    data_access_volume: int

class BurnoutRequest(BaseModel):
    user_id: str
    hours_worked_week: float
    tasks_completed: int
    tasks_overdue: int
    avg_task_completion_time: Optional[float] = 0
    tickets_raised: Optional[int] = 0

class AnomalyRequest(BaseModel):
    user_id: str
    action: str
    timestamp: str
    data_volume: Optional[int] = 0
    ip_address: str
    device: str

# Endpoints
@app.get("/")
async def root():
    return {
        "service": "Enterprise User Management ML Service",
        "version": "1.0.0",
        "status": "active"
    }

@app.post("/classify-ticket")
async def classify_ticket(request: TicketRequest):
    """Classify support ticket and suggest routing"""
    try:
        result = ticket_classifier.classify(
            request.subject,
            request.description
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-risk")
async def detect_risk(request: RiskRequest):
    """Detect user risk based on behavior patterns"""
    try:
        result = risk_detector.analyze({
            'user_id': request.user_id,
            'login_attempts': request.login_attempts,
            'failed_logins': request.failed_logins,
            'unusual_access_time': request.unusual_access_time,
            'location_changed': request.location_changed,
            'data_access_volume': request.data_access_volume
        })
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutRequest):
    """Analyze user burnout risk"""
    try:
        result = burnout_analyzer.analyze({
            'user_id': request.user_id,
            'hours_worked_week': request.hours_worked_week,
            'tasks_completed': request.tasks_completed,
            'tasks_overdue': request.tasks_overdue,
            'avg_task_completion_time': request.avg_task_completion_time,
            'tickets_raised': request.tickets_raised
        })
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyRequest):
    """Detect anomalous user behavior"""
    try:
        from models.anomaly_detector import AnomalyDetector
        anomaly_detector = AnomalyDetector()
        
        result = anomaly_detector.detect({
            'user_id': request.user_id,
            'action': request.action,
            'timestamp': request.timestamp,
            'data_volume': request.data_volume,
            'ip_address': request.ip_address,
            'device': request.device
        })
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "models_loaded": True}
```

## Configuration

### Environment Variables

**Backend (.env)**:
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=${JWT_SECRET}  # Use strong random string
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**:
```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
CORS_ORIGINS=http://localhost:3000,http://localhost:5000
CACHE_PREDICTIONS=true
MAX_BATCH_SIZE=100
```

**Frontend (.env)**:
```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_AI_FEATURES=true
```

### MongoDB Schema Examples

```javascript
// User Schema
const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive', 'suspended'], default: 'active' },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now }
});

// Task Schema
const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 },
  completedAt: Date,
  createdAt: { type: Date, default: Date.now }
});

// Ticket Schema
const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  category: { type: String, enum: ['technical', 'billing', 'access', 'general'] },
  priority: { type: String, enum: ['low', 'medium', 'high', 'critical'] },
  status: { type: String, enum: ['open', 'in-progress', 'resolved', 'closed'], default: 'open' },
  aiClassified: { type: Boolean, default: false },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});
```

## Common Patterns

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      throw new Error('No authentication token');
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');

    if (!user || user.status !== 'active') {
      throw new Error('User not found or inactive');
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
  }
};

// Admin-only middleware
const adminAuth = async (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { auth, adminAuth };
```

### Real-time AI Insights Integration

```javascript
// frontend/src/hooks/useAIInsights.js
import { useState, useEffect } from 'react';
import axios from 'axios';

export const useAIInsights = (userId) => {
  const [insights, setInsights] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchInsights = async () => {
      try {
        // Fetch user activity data
        const activityRes = await axios.get(
          `${process.env.REACT_APP_API_URL}/users/${userId}/activity`,
          { headers: { Authorization: `Bearer ${localStorage.getItem('token')}` } }
        );

        // Get AI analysis
        const [burnout, risk] = await Promise.all([
          axios.post(`${process.env.REACT_APP_ML_API_URL}/detect-burnout`, {
            user_id: userId,
            ...activityRes.data.workloadMetrics
          }),
          axios.post(`${process.env.REACT_APP_ML_API_URL}/detect-risk`, {
            user_id: userId,
            ...activityRes.data.securityMetrics
          })
        ]);

        setInsights({
          burnout: burnout.data,
          risk: risk.data,
          lastUpdated: new Date()
        });
      } catch (error) {
        console.error('Failed to fetch AI insights:', error);
      } finally {
        setLoading(false);
      }
    };

    fetchInsights();
    // Refresh every 5 minutes
    const interval = setInterval(fetchInsights, 5 * 60 * 1000);
    return () => clearInterval(interval);
  }, [userId]);

  return { insights, loading };
};
```

### Audit Logging Pattern

```javascript
// backend/utils/auditLog.js
const AuditLog = require('../models/AuditLog');

const logAuditEvent = async ({
  userId,
  action,
  resource,
  resourceId,
  changes = {},
  ipAddress,
  userAgent
}) => {
  try {
    await AuditLog.create({
      userId,
      action,
      resource,
      resourceId,
      changes,
      ipAddress,
      userAgent,
      timestamp: new Date()
    });
  } catch (error) {
    console.error('Audit log failed:', error);
    // Don't throw - logging shouldn't break main flow
  }
};

module.exports = { logAuditEvent };

// Usage in controllers
const { logAuditEvent } = require('../utils/auditLog');

exports.deleteUser = async (req, res) => {
  const user = await User.findById(req.params.id);
  await user.remove();
  
  await logAuditEvent({
    userId: req.user.id,
    action: 'USER_DELETED',
    resource: 'user',
    resourceId: req.params.id,
    changes: { deletedUser: user.email },
    ipAddress: req.ip,
    userAgent: req.get('user-agent')
  });
  
  res.json({ message: 'User deleted' });
};
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Check connection in backend
# Add to backend/index.js:
mongoose.connection.on('error', err => {
  console.error('MongoDB connection error:', err);
});

mongoose.connection.once('open', () => {
  console.log('✅ MongoDB connected successfully');
});
```

### ML Service Model Loading

```python
# ml-service/models/ticket_classifier.py
import logging

logger = logging.getLogger(__name__)

class TicketClassifier:
    def __init__(self, model_path='./models'):
        self.
