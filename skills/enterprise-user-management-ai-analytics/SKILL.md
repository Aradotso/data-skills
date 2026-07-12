---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create task tracking with burnout detection"
  - "build intelligent ticket routing system"
  - "add role-based access control with JWT"
  - "integrate ML service for risk prediction"
  - "configure kanban board with AI insights"
  - "deploy user management with anomaly detection"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-driven insights. It provides:

- **User Management**: Secure authentication, role-based access control, and user lifecycle management
- **Task Tracking**: Kanban boards, time tracking, and workload monitoring
- **Support System**: Intelligent ticket classification and automated routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay forecasting
- **Admin Dashboard**: Organization-wide analytics, audit logs, and alerts

The system consists of three main components:
1. React frontend (port 3000)
2. Node.js/Express backend (port 5000)
3. FastAPI ML service (port 8000)

## Installation

### Prerequisites

Ensure you have installed:
- Node.js 16+ and npm
- Python 3.8+
- MongoDB (local or Atlas)

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
NODE_ENV=development
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
REACT_APP_ML_URL=http://localhost:8000
EOF

npm start
```

## Key Components

### Backend API Endpoints

#### Authentication
```javascript
// User registration
POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "securePass123",
  "role": "user"
}

// User login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePass123"
}
// Response: { "token": "jwt_token", "user": {...} }
```

#### User Management
```javascript
// Get all users (Admin only)
GET /api/users
Headers: { Authorization: "Bearer ${JWT_TOKEN}" }

// Update user
PUT /api/users/:id
{
  "username": "john.updated",
  "role": "manager"
}

// Delete user (Admin only)
DELETE /api/users/:id
```

#### Task Management
```javascript
// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Add new dashboard component",
  "assignedTo": "user_id",
  "status": "todo",
  "priority": "high",
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:id/time
{
  "duration": 3600, // seconds
  "date": "2026-04-15"
}
```

#### Support Tickets
```javascript
// Create ticket
POST /api/tickets
{
  "subject": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// Get tickets (filtered)
GET /api/tickets?status=open&priority=high
```

### ML Service API Endpoints

#### Risk Prediction
```python
# POST /api/ml/predict-risk
{
  "userId": "user_id",
  "features": {
    "tasksCompleted": 15,
    "averageCompletionTime": 4.2,
    "overdueCount": 2,
    "loginFrequency": 0.9,
    "errorRate": 0.05
  }
}
# Response: { "riskScore": 0.35, "riskLevel": "medium", "factors": [...] }
```

#### Anomaly Detection
```python
# POST /api/ml/detect-anomaly
{
  "userId": "user_id",
  "behavior": {
    "loginTime": "03:00",
    "location": "unknown",
    "actionsPerMinute": 50,
    "dataAccessed": ["sensitive_file_1", "sensitive_file_2"]
  }
}
# Response: { "isAnomaly": true, "confidence": 0.87, "anomalyType": "unusual_time" }
```

#### Burnout Detection
```python
# POST /api/ml/burnout-analysis
{
  "userId": "user_id",
  "workload": {
    "hoursWorked": 55,
    "tasksAssigned": 12,
    "tasksCompleted": 5,
    "missedDeadlines": 3,
    "weekendWork": true
  }
}
# Response: { "burnoutScore": 0.72, "level": "high", "recommendations": [...] }
```

#### Ticket Classification
```python
# POST /api/ml/classify-ticket
{
  "subject": "Cannot access database",
  "description": "Getting connection timeout errors when trying to connect to production DB",
  "userRole": "developer"
}
# Response: { "category": "technical", "priority": "high", "suggestedAssignee": "support_team_lead" }
```

## Code Examples

### Frontend: Creating a Task with AI Insights

```javascript
// src/components/TaskCreator.jsx
import React, { useState } from 'react';
import axios from 'axios';

const TaskCreator = ({ userId }) => {
  const [task, setTask] = useState({
    title: '',
    description: '',
    assignedTo: '',
    priority: 'medium',
    dueDate: ''
  });
  const [aiInsights, setAiInsights] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    try {
      // Create task
      const token = localStorage.getItem('token');
      const taskResponse = await axios.post(
        `${process.env.REACT_APP_API_URL}/api/tasks`,
        task,
        { headers: { Authorization: `Bearer ${token}` } }
      );

      // Get AI prediction for assigned user
      const mlResponse = await axios.post(
        `${process.env.REACT_APP_ML_URL}/api/ml/predict-risk`,
        {
          userId: task.assignedTo,
          features: {
            newTaskPriority: task.priority,
            dueDate: task.dueDate
          }
        }
      );

      setAiInsights(mlResponse.data);
      
      if (mlResponse.data.riskLevel === 'high') {
        alert('Warning: Assigned user shows high burnout risk!');
      }

      console.log('Task created:', taskResponse.data);
    } catch (error) {
      console.error('Error creating task:', error);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        placeholder="Task Title"
        value={task.title}
        onChange={(e) => setTask({ ...task, title: e.target.value })}
        required
      />
      <textarea
        placeholder="Description"
        value={task.description}
        onChange={(e) => setTask({ ...task, description: e.target.value })}
      />
      <select
        value={task.priority}
        onChange={(e) => setTask({ ...task, priority: e.target.value })}
      >
        <option value="low">Low</option>
        <option value="medium">Medium</option>
        <option value="high">High</option>
      </select>
      <input
        type="date"
        value={task.dueDate}
        onChange={(e) => setTask({ ...task, dueDate: e.target.value })}
      />
      <button type="submit">Create Task</button>
      
      {aiInsights && (
        <div className="ai-insights">
          <h4>AI Risk Assessment</h4>
          <p>Risk Score: {aiInsights.riskScore}</p>
          <p>Risk Level: {aiInsights.riskLevel}</p>
        </div>
      )}
    </form>
  );
};

export default TaskCreator;
```

### Backend: Task Controller with AI Integration

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const User = require('../models/User');
const axios = require('axios');

// Create new task
exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;

    // Validate assigned user exists
    const user = await User.findById(assignedTo);
    if (!user) {
      return res.status(404).json({ message: 'Assigned user not found' });
    }

    // Create task
    const task = await Task.create({
      title,
      description,
      assignedTo,
      createdBy: req.user.id,
      priority,
      dueDate,
      status: 'todo'
    });

    // Get AI burnout prediction for assigned user
    try {
      const userTasks = await Task.find({ assignedTo }).countDocuments();
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/ml/burnout-analysis`,
        {
          userId: assignedTo,
          workload: {
            tasksAssigned: userTasks + 1,
            newTaskPriority: priority
          }
        }
      );

      if (mlResponse.data.burnoutScore > 0.7) {
        // Send notification to admin
        console.warn(`High burnout risk for user ${assignedTo}`);
      }
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }

    res.status(201).json({ success: true, task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Get tasks with filtering
exports.getTasks = async (req, res) => {
  try {
    const { status, priority, assignedTo } = req.query;
    const filter = {};

    if (status) filter.status = status;
    if (priority) filter.priority = priority;
    if (assignedTo) filter.assignedTo = assignedTo;

    // Non-admin users only see their tasks
    if (req.user.role !== 'admin') {
      filter.assignedTo = req.user.id;
    }

    const tasks = await Task.find(filter)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username')
      .sort({ createdAt: -1 });

    res.json({ success: true, tasks });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Update task status
exports.updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;

    const task = await Task.findById(id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    // Check authorization
    if (req.user.role !== 'admin' && task.assignedTo.toString() !== req.user.id) {
      return res.status(403).json({ message: 'Not authorized' });
    }

    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    await task.save();

    res.json({ success: true, task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### ML Service: Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
import os

class RiskPredictor:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        self.model = None
        self.feature_names = [
            'tasks_completed',
            'avg_completion_time',
            'overdue_count',
            'login_frequency',
            'error_rate'
        ]
        
        if os.path.exists(model_path):
            self.load_model()
        else:
            self.train_initial_model()
    
    def train_initial_model(self):
        """Train initial model with synthetic data"""
        # Generate synthetic training data
        np.random.seed(42)
        n_samples = 1000
        
        X = np.random.rand(n_samples, 5)
        # Risk score based on features
        y = (X[:, 2] > 0.5) | (X[:, 4] > 0.6) | (X[:, 3] < 0.3)
        
        self.model = RandomForestClassifier(n_estimators=100, random_state=42)
        self.model.fit(X, y)
        self.save_model()
    
    def predict(self, features):
        """Predict risk level for user"""
        # Prepare features
        X = np.array([[
            features.get('tasksCompleted', 0),
            features.get('averageCompletionTime', 0),
            features.get('overdueCount', 0),
            features.get('loginFrequency', 0),
            features.get('errorRate', 0)
        ]])
        
        # Get prediction and probability
        risk_binary = self.model.predict(X)[0]
        risk_proba = self.model.predict_proba(X)[0]
        
        risk_score = risk_proba[1] if len(risk_proba) > 1 else 0.0
        
        # Determine risk level
        if risk_score < 0.3:
            risk_level = 'low'
        elif risk_score < 0.6:
            risk_level = 'medium'
        else:
            risk_level = 'high'
        
        # Get feature importance
        feature_importance = dict(zip(
            self.feature_names,
            self.model.feature_importances_
        ))
        
        return {
            'riskScore': float(risk_score),
            'riskLevel': risk_level,
            'factors': sorted(
                feature_importance.items(),
                key=lambda x: x[1],
                reverse=True
            )[:3]
        }
    
    def save_model(self):
        """Save model to disk"""
        os.makedirs(os.path.dirname(self.model_path), exist_ok=True)
        joblib.dump(self.model, self.model_path)
    
    def load_model(self):
        """Load model from disk"""
        self.model = joblib.load(self.model_path)

# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Dict, Any
from models.risk_predictor import RiskPredictor
import uvicorn

app = FastAPI(title="Enterprise User Management ML Service")

# Initialize models
risk_predictor = RiskPredictor()

class RiskPredictionRequest(BaseModel):
    userId: str
    features: Dict[str, Any]

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    workload: Dict[str, Any]

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str
    userRole: str

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavioral features"""
    try:
        result = risk_predictor.predict(request.features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze user burnout risk based on workload"""
    try:
        workload = request.workload
        
        # Calculate burnout score
        hours_factor = min(workload.get('hoursWorked', 40) / 40, 2.0)
        task_ratio = workload.get('tasksAssigned', 0) / max(workload.get('tasksCompleted', 1), 1)
        deadline_factor = workload.get('missedDeadlines', 0) * 0.2
        weekend_factor = 0.3 if workload.get('weekendWork', False) else 0
        
        burnout_score = min(
            (hours_factor * 0.3 + task_ratio * 0.3 + deadline_factor + weekend_factor) / 2,
            1.0
        )
        
        if burnout_score < 0.4:
            level = 'low'
            recommendations = ['Maintain current workload balance']
        elif burnout_score < 0.7:
            level = 'medium'
            recommendations = [
                'Consider redistributing some tasks',
                'Monitor overtime hours'
            ]
        else:
            level = 'high'
            recommendations = [
                'Immediate workload reduction needed',
                'Schedule time off',
                'Redistribute urgent tasks'
            ]
        
        return {
            'burnoutScore': float(burnout_score),
            'level': level,
            'recommendations': recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket and suggest routing"""
    try:
        subject = request.subject.lower()
        description = request.description.lower()
        text = f"{subject} {description}"
        
        # Simple rule-based classification
        if any(word in text for word in ['database', 'connection', 'query', 'sql']):
            category = 'technical'
            priority = 'high'
            assignee = 'database_team'
        elif any(word in text for word in ['login', 'password', 'access', 'permission']):
            category = 'access'
            priority = 'high'
            assignee = 'security_team'
        elif any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
            priority = 'medium'
            assignee = 'support_team'
        else:
            category = 'general'
            priority = 'low'
            assignee = 'support_team'
        
        return {
            'category': category,
            'priority': priority,
            'suggestedAssignee': assignee
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=your-secret-key-min-32-chars
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
MODEL_PATH=./models
LOG_LEVEL=INFO
RETRAIN_INTERVAL=86400
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_ENV=development
```

### MongoDB Setup

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

## Common Patterns

### Protected Routes with JWT

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization?.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Token invalid' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: 'Not authorized for this role' });
    }
    next();
  };
};

// Usage in routes
router.get('/admin/users', protect, authorize('admin'), getUsers);
```

### Real-time AI Monitoring

```javascript
// backend/services/monitoringService.js
const axios = require('axios');
const User = require('../models/User');

class MonitoringService {
  constructor() {
    this.mlServiceUrl = process.env.ML_SERVICE_URL;
  }

  async checkUserAnomaly(userId, behavior) {
    try {
      const response = await axios.post(
        `${this.mlServiceUrl}/api/ml/detect-anomaly`,
        { userId, behavior }
      );

      if (response.data.isAnomaly && response.data.confidence > 0.8) {
        await this.logSecurityEvent(userId, response.data);
        await this.notifyAdmin(userId, response.data);
      }

      return response.data;
    } catch (error) {
      console.error('Anomaly detection error:', error);
      return null;
    }
  }

  async logSecurityEvent(userId, anomaly) {
    // Log to audit trail
    console.log(`Security event for user ${userId}:`, anomaly);
  }

  async notifyAdmin(userId, anomaly) {
    // Send notification to admin
    console.log(`Admin notification: Anomaly detected for user ${userId}`);
  }
}

module.exports = new MonitoringService();
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Verify connection
mongosh --eval "db.adminCommand('ping')"
```

### ML Service Not Starting

```bash
# Check Python version
python --version  # Should be 3.8+

# Reinstall dependencies
pip install --upgrade -r ml-service/requirements.txt

# Check port availability
lsof -i :8000

# Run with verbose logging
cd ml-service
LOG_LEVEL=DEBUG uvicorn main:app --reload
```

### JWT Token Errors

```javascript
// Verify token format
const token = localStorage.getItem('token');
console.log('Token:', token);

// Check token expiration
const decoded = jwt.decode(token);
console.log('Expires:', new Date(decoded.exp * 1000));

// Refresh token if expired
if (Date.now() >= decoded.exp * 1000) {
  // Redirect to login
}
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### Model Not Found Error

```bash
# Create models directory
mkdir -p ml-service/models

# Retrain model
cd ml-service
python -c "from models.risk_predictor import RiskPredictor; RiskPredictor()"
```

### Task Status Not Updating

```javascript
// Check task ownership
const task = await Task.findById(taskId);
console.log('Assigned to:', task.assignedTo);
console.log('Current user:', req.user.id);

// Verify authorization
if (task.assignedTo.toString() !== req.user.id && req.user.role !== 'admin') {
  throw new Error('Not authorized to update this task');
}
```
