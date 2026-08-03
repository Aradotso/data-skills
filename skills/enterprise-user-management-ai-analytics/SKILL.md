---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, task automation, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management with task tracking"
  - "build admin dashboard with AI insights"
  - "add AI ticket classification system"
  - "integrate burnout detection for users"
  - "deploy user management with ML service"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack JavaScript application that combines traditional user/task management with AI-powered analytics. It features role-based access control (admin/user), Kanban task boards, support ticket management, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and predictive project delays.

**Stack:**
- Frontend: React.js
- Backend: Node.js with Express
- ML Service: FastAPI (Python) with scikit-learn and River
- Database: MongoDB
- Auth: JWT

## Installation

### Prerequisites

```bash
# Node.js 14+, Python 3.8+, MongoDB installed/accessible
node --version
python --version
mongod --version
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=info
EOF

uvicorn main:app --reload --host 0.0.0.0 --port 8000
# ML service runs at http://localhost:8000
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
# Frontend runs at http://localhost:3000
```

## Architecture

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   React      │─────▶│   Node.js    │─────▶│   MongoDB    │
│   Frontend   │      │   Backend    │      │   Database   │
└──────────────┘      └──────┬───────┘      └──────────────┘
                             │
                             ▼
                      ┌──────────────┐
                      │   FastAPI    │
                      │  ML Service  │
                      └──────────────┘
```

## Key Features & Code Examples

### 1. User Authentication (Backend)

**JWT Token Generation:**

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const generateToken = (userId, role) => {
  return jwt.sign(
    { userId, role },
    process.env.JWT_SECRET,
    { expiresIn: '24h' }
  );
};

const verifyToken = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'Access denied' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

module.exports = { generateToken, verifyToken };
```

**User Login Route:**

```javascript
// backend/routes/auth.js
const express = require('express');
const bcrypt = require('bcryptjs');
const User = require('../models/User');
const { generateToken } = require('../middleware/auth');

const router = express.Router();

router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = generateToken(user._id, user.role);
    
    res.json({
      token,
      user: {
        id: user._id,
        email: user.email,
        role: user.role,
        name: user.name
      }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### 2. Task Management (Frontend)

**Kanban Board Component:**

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'inProgress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div 
              key={task._id} 
              className="task-card"
              draggable
              onDragStart={(e) => e.dataTransfer.setData('taskId', task._id)}
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className="priority">{task.priority}</span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### 3. AI Risk Detection (ML Service)

**Risk Prediction Model:**

```python
# ml-service/models/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
import os

class RiskPredictor:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        if os.path.exists(model_path):
            self.model = joblib.load(model_path)
        else:
            self.model = RandomForestClassifier(n_estimators=100, random_state=42)
    
    def extract_features(self, user_data):
        """Extract features from user activity data"""
        features = [
            user_data.get('failed_login_attempts', 0),
            user_data.get('after_hours_access', 0),
            user_data.get('data_download_volume', 0),
            user_data.get('privilege_escalation_attempts', 0),
            user_data.get('unusual_ip_access', 0),
            user_data.get('inactive_days', 0),
            user_data.get('avg_session_duration', 0)
        ]
        return np.array(features).reshape(1, -1)
    
    def predict_risk(self, user_data):
        """Predict risk level (0: low, 1: medium, 2: high)"""
        features = self.extract_features(user_data)
        risk_level = self.model.predict(features)[0]
        risk_prob = self.model.predict_proba(features)[0]
        
        return {
            'risk_level': int(risk_level),
            'confidence': float(max(risk_prob)),
            'probabilities': {
                'low': float(risk_prob[0]),
                'medium': float(risk_prob[1]) if len(risk_prob) > 1 else 0,
                'high': float(risk_prob[2]) if len(risk_prob) > 2 else 0
            }
        }
    
    def train(self, X, y):
        """Train the model with labeled data"""
        self.model.fit(X, y)
        joblib.dump(self.model, self.model_path)
```

**FastAPI Endpoint:**

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector
from models.anomaly_detector import AnomalyDetector

app = FastAPI(title="Enterprise ML Service")

risk_predictor = RiskPredictor()
burnout_detector = BurnoutDetector()
anomaly_detector = AnomalyDetector()

class UserActivityData(BaseModel):
    user_id: str
    failed_login_attempts: int = 0
    after_hours_access: int = 0
    data_download_volume: float = 0
    privilege_escalation_attempts: int = 0
    unusual_ip_access: int = 0
    inactive_days: int = 0
    avg_session_duration: float = 0

class TaskWorkload(BaseModel):
    user_id: str
    total_tasks: int
    overdue_tasks: int
    avg_completion_time: float
    working_hours_per_week: float
    weekend_work_hours: float
    task_completion_rate: float

@app.post("/api/ml/risk-prediction")
async def predict_risk(data: UserActivityData):
    try:
        result = risk_predictor.predict_risk(data.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(data: TaskWorkload):
    try:
        result = burnout_detector.analyze(data.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(data: UserActivityData):
    try:
        result = anomaly_detector.detect(data.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

### 4. Burnout Detection

```python
# ml-service/models/burnout_detector.py
import numpy as np
from river import tree, compose, preprocessing

class BurnoutDetector:
    def __init__(self):
        # Online learning model using River
        self.model = compose.Pipeline(
            preprocessing.StandardScaler(),
            tree.HoeffdingTreeClassifier()
        )
        self.is_trained = False
    
    def extract_features(self, workload_data):
        """Extract burnout risk features"""
        return {
            'task_overload': workload_data.get('total_tasks', 0) / 10,
            'overdue_ratio': (
                workload_data.get('overdue_tasks', 0) / 
                max(workload_data.get('total_tasks', 1), 1)
            ),
            'overtime_hours': max(
                workload_data.get('working_hours_per_week', 40) - 40, 0
            ),
            'weekend_work': workload_data.get('weekend_work_hours', 0),
            'completion_pressure': 1 - workload_data.get('task_completion_rate', 0.7)
        }
    
    def analyze(self, workload_data):
        """Analyze burnout risk"""
        features = self.extract_features(workload_data)
        
        # Rule-based scoring for interpretability
        burnout_score = 0
        factors = []
        
        if features['task_overload'] > 5:
            burnout_score += 0.3
            factors.append("High task volume")
        
        if features['overdue_ratio'] > 0.2:
            burnout_score += 0.25
            factors.append("Many overdue tasks")
        
        if features['overtime_hours'] > 10:
            burnout_score += 0.25
            factors.append("Excessive overtime")
        
        if features['weekend_work'] > 5:
            burnout_score += 0.15
            factors.append("Weekend work pattern")
        
        if features['completion_pressure'] > 0.4:
            burnout_score += 0.05
            factors.append("Low completion rate")
        
        risk_level = "low"
        if burnout_score > 0.7:
            risk_level = "high"
        elif burnout_score > 0.4:
            risk_level = "medium"
        
        return {
            'burnout_score': round(burnout_score, 2),
            'risk_level': risk_level,
            'contributing_factors': factors,
            'recommendations': self._get_recommendations(risk_level, factors)
        }
    
    def _get_recommendations(self, risk_level, factors):
        """Generate actionable recommendations"""
        recommendations = []
        
        if risk_level == "high":
            recommendations.append("Immediate workload redistribution recommended")
            recommendations.append("Schedule 1-on-1 with manager")
        
        if "High task volume" in factors:
            recommendations.append("Consider delegating or postponing lower priority tasks")
        
        if "Excessive overtime" in factors or "Weekend work pattern" in factors:
            recommendations.append("Enforce work-life balance policies")
        
        if "Many overdue tasks" in factors:
            recommendations.append("Review task deadlines and priorities")
        
        return recommendations
```

### 5. Support Ticket Classification

```python
# ml-service/models/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
import joblib
import os

class TicketClassifier:
    def __init__(self, model_path='./models/ticket_classifier.pkl'):
        self.model_path = model_path
        
        if os.path.exists(model_path):
            self.pipeline = joblib.load(model_path)
        else:
            self.pipeline = Pipeline([
                ('tfidf', TfidfVectorizer(max_features=1000)),
                ('classifier', MultinomialNB())
            ])
        
        self.categories = ['Technical', 'HR', 'Access', 'General']
        self.priority_keywords = {
            'critical': ['urgent', 'critical', 'down', 'emergency', 'broken'],
            'high': ['important', 'asap', 'blocking', 'cannot'],
            'medium': ['need', 'should', 'help', 'issue'],
            'low': ['question', 'request', 'when possible']
        }
    
    def classify_ticket(self, ticket_text):
        """Classify ticket category and priority"""
        # Predict category
        category = self.pipeline.predict([ticket_text])[0]
        
        # Determine priority based on keywords
        priority = self._determine_priority(ticket_text)
        
        # Route to appropriate department
        routing = self._route_ticket(category, priority)
        
        return {
            'category': category,
            'priority': priority,
            'assigned_department': routing['department'],
            'estimated_response_time': routing['response_time'],
            'confidence': 0.85  # Add actual confidence from model
        }
    
    def _determine_priority(self, text):
        """Determine ticket priority from text"""
        text_lower = text.lower()
        
        for priority, keywords in self.priority_keywords.items():
            if any(keyword in text_lower for keyword in keywords):
                return priority
        
        return 'medium'
    
    def _route_ticket(self, category, priority):
        """Route ticket to appropriate team"""
        routing_map = {
            'Technical': {
                'department': 'IT Support',
                'response_time': '2 hours' if priority in ['critical', 'high'] else '24 hours'
            },
            'HR': {
                'department': 'Human Resources',
                'response_time': '4 hours' if priority == 'critical' else '48 hours'
            },
            'Access': {
                'department': 'Security Team',
                'response_time': '1 hour' if priority == 'critical' else '4 hours'
            },
            'General': {
                'department': 'General Support',
                'response_time': '24 hours'
            }
        }
        
        return routing_map.get(category, routing_map['General'])
    
    def train(self, tickets, labels):
        """Train classifier with ticket data"""
        self.pipeline.fit(tickets, labels)
        joblib.dump(self.pipeline, self.model_path)
```

### 6. Admin Dashboard Integration (Backend)

**Analytics Endpoint:**

```javascript
// backend/routes/analytics.js
const express = require('express');
const axios = require('axios');
const User = require('../models/User');
const Task = require('../models/Task');
const { verifyToken } = require('../middleware/auth');

const router = express.Router();

router.get('/dashboard', verifyToken, async (req, res) => {
  try {
    // Only admins can access analytics
    if (req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Access denied' });
    }
    
    // Get user statistics
    const totalUsers = await User.countDocuments();
    const activeUsers = await User.countDocuments({ 
      lastLogin: { $gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) } 
    });
    
    // Get task statistics
    const totalTasks = await Task.countDocuments();
    const completedTasks = await Task.countDocuments({ status: 'done' });
    const overdueTasks = await Task.countDocuments({ 
      dueDate: { $lt: new Date() },
      status: { $ne: 'done' }
    });
    
    // Get AI insights for each user
    const users = await User.find().select('-password');
    const aiInsights = await Promise.all(
      users.map(async (user) => {
        try {
          const userTasks = await Task.find({ assignedTo: user._id });
          
          // Call ML service for risk prediction
          const riskResponse = await axios.post(
            `${process.env.ML_SERVICE_URL}/api/ml/risk-prediction`,
            {
              user_id: user._id,
              failed_login_attempts: user.failedLoginAttempts || 0,
              after_hours_access: user.afterHoursAccess || 0,
              inactive_days: user.inactiveDays || 0
            }
          );
          
          // Call ML service for burnout detection
          const burnoutResponse = await axios.post(
            `${process.env.ML_SERVICE_URL}/api/ml/burnout-detection`,
            {
              user_id: user._id,
              total_tasks: userTasks.length,
              overdue_tasks: userTasks.filter(t => t.dueDate < new Date() && t.status !== 'done').length,
              working_hours_per_week: user.workingHours || 40,
              weekend_work_hours: user.weekendHours || 0,
              task_completion_rate: user.completionRate || 0.7
            }
          );
          
          return {
            userId: user._id,
            name: user.name,
            risk: riskResponse.data,
            burnout: burnoutResponse.data
          };
        } catch (error) {
          console.error(`Error getting insights for user ${user._id}:`, error.message);
          return null;
        }
      })
    );
    
    res.json({
      statistics: {
        totalUsers,
        activeUsers,
        totalTasks,
        completedTasks,
        overdueTasks,
        completionRate: totalTasks > 0 ? (completedTasks / totalTasks * 100).toFixed(2) : 0
      },
      aiInsights: aiInsights.filter(insight => insight !== null),
      alerts: aiInsights
        .filter(i => i && (i.risk.risk_level > 1 || i.burnout.risk_level === 'high'))
        .map(i => ({
          userId: i.userId,
          name: i.name,
          type: i.risk.risk_level > 1 ? 'security_risk' : 'burnout_risk',
          severity: i.risk.risk_level > 1 ? 'high' : i.burnout.risk_level
        }))
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## Common Workflows

### Creating a New User (Admin)

```javascript
// Frontend: src/components/admin/CreateUser.jsx
import axios from 'axios';

const createUser = async (userData) => {
  const token = localStorage.getItem('token');
  
  const response = await axios.post(
    `${process.env.REACT_APP_API_URL}/api/users`,
    {
      name: userData.name,
      email: userData.email,
      role: userData.role, // 'admin' or 'user'
      department: userData.department
    },
    {
      headers: { Authorization: `Bearer ${token}` }
    }
  );
  
  return response.data;
};
```

### Raising a Support Ticket

```javascript
// Frontend: src/components/user/CreateTicket.jsx
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  
  const response = await axios.post(
    `${process.env.REACT_APP_API_URL}/api/tickets`,
    {
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority // Optional, AI will classify
    },
    {
      headers: { Authorization: `Bearer ${token}` }
    }
  );
  
  // Backend automatically calls ML service for classification
  return response.data;
};
```

### Task Time Tracking

```javascript
// Frontend: src/components/user/TaskTimer.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskTimer = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const startTimer = () => setIsRunning(true);
  
  const stopTimer = async () => {
    setIsRunning(false);
    
    // Save time entry
    const token = localStorage.getItem('token');
    await axios.post(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`,
      { duration: seconds },
      { headers: { Authorization: `Bearer ${token}` } }
    );
  };

  return (
    <div>
      <p>Time: {Math.floor(seconds / 3600)}h {Math.floor((seconds % 3600) / 60)}m {seconds % 60}s</p>
      <button onClick={startTimer} disabled={isRunning}>Start</button>
      <button onClick={stopTimer} disabled={!isRunning}>Stop</button>
    </div>
  );
};
```

## Configuration

### Backend Environment Variables

```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your-secret-key-here  # Use ${JWT_SECRET} env var
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
SESSION_TIMEOUT=86400000  # 24 hours in ms
MAX_LOGIN_ATTEMPTS=5
```

### ML Service Configuration

```bash
# ml-service/.env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=info
RETRAIN_INTERVAL=86400  # Retrain models every 24 hours
MIN_TRAINING_SAMPLES=100
```

### Frontend Configuration

```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_ANALYTICS=true
REACT_APP_SESSION_TIMEOUT=86400000
```

## MongoDB Schema Examples

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  department: { type: String },
  failedLoginAttempts: { type: Number, default: 0 },
  lastLogin: { type: Date },
  createdAt: { type: Date, default: Date.now },
  isActive: { type: Boolean, default: true },
  
  // Tracking fields for AI
  afterHoursAccess: { type: Number, default: 0 },
  inactiveDays: { type: Number, default: 0 },
  workingHours: { type: Number, default: 40 },
  weekendHours: { type: Number, default: 0 },
  completionRate: { type: Number, default: 0.7 }
});

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  status: { 
    type: String, 
    enum: ['todo', 'inProgress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  dueDate: { type: Date },
  createdAt: { type: Date, default: Date.now },
  completedAt: { type: Date },
  timeSpent: { type: Number, default: 0 }, // in seconds
  tags: [{ type: String }]
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { 
    type: String, 
    enum: ['Technical', 'HR', 'Access', 'General'] 
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
  raisedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: String }, // Department name
  createdAt: { type: Date, default: Date.now },
  resolvedAt: { type: Date },
  aiClassification: {
    category: String,
    priority: String,
    confidence: Number
  }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## API Endpoints Reference

### Authentication
- `POST /api/auth/login` - User login
- `POST /api/auth/register` - User registration (admin only)
- `POST /api/auth/logout` - User logout

### Users
- `GET /api/users` - Get all users (admin only)
- `GET /api/users/:id` - Get user by ID
- `POST /api/users` - Create new user (admin only)
- `PATCH /api/users/:id` - Update user
- `DELETE /api/users/:id` - Delete user (admin only)

### Tasks
- `GET /api/tasks` - Get tasks for logged-in user
- `POST /api/tasks` - Create new task
- `PATCH /api/tasks/:id` - Update task
- `DELETE /api/tasks/:id` - Delete task
- `POST /api/tasks/:id/time` - Log time spent

### Tickets
- `GET /api/tickets` - Get all
