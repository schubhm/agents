# AI-Powered Business Intelligence Chat System
## Project Plan for acme corporation BI Transformation

**Project Lead:** Director of BI  
**Objective:** Transform traditional dashboard delivery to AI-powered conversational analytics  
**Timeline:** 12-16 weeks  
**Last Updated:** August 2025

---

## Executive Summary

This project will develop an AI-powered chat system that replaces traditional dashboard requests with natural language queries. The system will automatically generate SQL, identify appropriate databases, execute queries, and return results as tables and visualizations through a conversational interface.

---

## Project Architecture Overview

### Core Components
1. **Natural Language Processor** - Converts business questions to SQL
2. **Database Router** - Identifies correct database and tables
3. **Query Engine** - Executes SQL safely with proper permissions
4. **Visualization Engine** - Generates charts and tables from results
5. **Chat Interface** - User-friendly conversational UI
6. **Model Context Protocol (MCP)** - For database connections and tool orchestration

### Technology Stack
- **Backend:** Python (FastAPI/Flask)
- **AI/LLM:** OpenAI GPT-4 or Claude 3.5 Sonnet with function calling
- **Database Connectivity:** SQLAlchemy, database-specific drivers
- **Visualization:** Plotly, D3.js, or similar
- **Frontend:** React with chat UI components
- **MCP Server:** Custom MCP implementation for database access
- **Agent Framework:** LangChain or similar for orchestration

---

## Phase 1: Foundation & Planning (Weeks 1-2)

### Deliverables
- [ ] Database inventory and schema documentation
- [ ] Data governance and security requirements
- [ ] User persona and use case analysis
- [ ] Technical architecture finalization

### Key Activities
1. **Database Audit**
   - Catalog all data sources (data warehouse, operational DBs, etc.)
   - Document table relationships and business glossary
   - Map common business metrics to underlying tables

2. **Security Framework**
   - Define row-level security requirements
   - Establish query execution limits and timeouts
   - Create user permission matrix

---

## Phase 2: Core AI Agent Development (Weeks 3-6)

### 2.1 Natural Language to SQL Agent

```python
# ai_sql_agent.py
import openai
from typing import Dict, List, Optional
from dataclasses import dataclass
from sqlalchemy import create_engine, text
import json

@dataclass
class DatabaseSchema:
    """Represents database schema information"""
    database_name: str
    tables: Dict[str, Dict[str, str]]  # table_name -> {column: type}
    relationships: List[Dict]
    business_glossary: Dict[str, str]

class SQLAgent:
    def __init__(self, schemas: List[DatabaseSchema], llm_client):
        self.schemas = schemas
        self.llm_client = llm_client
        self.conversation_history = []
    
    def generate_sql(self, user_question: str, context: Dict = None) -> Dict:
        """Convert natural language to SQL with database routing"""
        
        # Create system prompt with schema information
        schema_info = self._format_schema_for_prompt()
        
        system_prompt = f"""
        You are an expert SQL generator for acme corporation's business intelligence system.
        
        Available databases and schemas:
        {schema_info}
        
        Rules:
        1. Always identify the correct database first
        2. Use proper table joins based on documented relationships
        3. Apply appropriate filters for data security
        4. Optimize queries for performance
        5. Return response in JSON format with: database, sql, explanation, expected_columns
        
        Business Context:
        - acme corporation is a radio/audio media company
        - Common metrics: audience, revenue, ratings, inventory
        - Always consider date ranges for time-series data
        """
        
        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"Question: {user_question}"}
        ]
        
        response = self.llm_client.chat.completions.create(
            model="gpt-4",
            messages=messages,
            functions=[
                {
                    "name": "generate_sql_query",
                    "description": "Generate SQL query with database routing",
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "database": {"type": "string"},
                            "sql": {"type": "string"},
                            "explanation": {"type": "string"},
                            "expected_columns": {"type": "array", "items": {"type": "string"}},
                            "visualization_type": {"type": "string", "enum": ["table", "bar_chart", "line_chart", "pie_chart"]}
                        },
                        "required": ["database", "sql", "explanation", "expected_columns"]
                    }
                }
            ],
            function_call={"name": "generate_sql_query"}
        )
        
        return json.loads(response.choices[0].message.function_call.arguments)
```

### 2.2 Database Router and Query Executor

```python
# database_router.py
from typing import Dict, Any
import pandas as pd
from sqlalchemy import create_engine, text
import logging

class DatabaseRouter:
    def __init__(self, connection_configs: Dict[str, str]):
        self.engines = {}
        self.connection_configs = connection_configs
        self._initialize_connections()
    
    def _initialize_connections(self):
        """Initialize database connections"""
        for db_name, connection_string in self.connection_configs.items():
            try:
                self.engines[db_name] = create_engine(
                    connection_string,
                    pool_pre_ping=True,
                    pool_recycle=3600
                )
                logging.info(f"Connected to database: {db_name}")
            except Exception as e:
                logging.error(f"Failed to connect to {db_name}: {e}")
    
    def execute_query(self, database: str, sql_query: str, user_permissions: Dict = None) -> pd.DataFrame:
        """Execute SQL query with security checks"""
        
        # Security validations
        if not self._validate_query_security(sql_query, user_permissions):
            raise SecurityError("Query not permitted for user")
        
        # Add row-level security filters if needed
        secured_query = self._apply_security_filters(sql_query, user_permissions)
        
        try:
            engine = self.engines.get(database)
            if not engine:
                raise ValueError(f"Database {database} not available")
            
            # Execute with timeout
            with engine.connect() as conn:
                result = pd.read_sql(text(secured_query), conn)
                logging.info(f"Query executed successfully: {len(result)} rows returned")
                return result
                
        except Exception as e:
            logging.error(f"Query execution failed: {e}")
            raise
    
    def _validate_query_security(self, sql_query: str, user_permissions: Dict) -> bool:
        """Validate query against security rules"""
        # Implement your security validation logic
        forbidden_keywords = ['DROP', 'DELETE', 'UPDATE', 'INSERT', 'ALTER']
        query_upper = sql_query.upper()
        
        return not any(keyword in query_upper for keyword in forbidden_keywords)
    
    def _apply_security_filters(self, sql_query: str, user_permissions: Dict) -> str:
        """Apply row-level security filters"""
        # Example: Add WHERE clauses based on user permissions
        # This would be customized based on your security model
        return sql_query
```

### 2.3 MCP Server Implementation

```python
# mcp_server.py
from mcp.server.models import InitializationOptions
from mcp.server import NotificationOptions, Server
from mcp.types import Resource, Tool, TextContent, ImageContent, EmbeddedResource
import asyncio
from typing import Any, Sequence

class BIAnalyticsMCPServer:
    def __init__(self):
        self.server = Server("bi-analytics")
        self.sql_agent = None
        self.db_router = None
        
        # Register tools
        self.server.list_tools = self.list_tools
        self.server.call_tool = self.call_tool
        
        # Register resources
        self.server.list_resources = self.list_resources
        self.server.read_resource = self.read_resource
    
    async def list_tools(self) -> list[Tool]:
        """List available tools for the MCP client"""
        return [
            Tool(
                name="execute_bi_query",
                description="Execute a business intelligence query in natural language",
                inputSchema={
                    "type": "object",
                    "properties": {
                        "question": {
                            "type": "string",
                            "description": "Natural language business question"
                        },
                        "user_context": {
                            "type": "object",
                            "description": "User permissions and context"
                        }
                    },
                    "required": ["question"]
                }
            ),
            Tool(
                name="get_database_schema",
                description="Get schema information for available databases",
                inputSchema={
                    "type": "object",
                    "properties": {
                        "database_name": {
                            "type": "string",
                            "description": "Name of the database"
                        }
                    }
                }
            )
        ]
    
    async def call_tool(self, name: str, arguments: dict) -> Sequence[TextContent | ImageContent | EmbeddedResource]:
        """Execute tool calls"""
        if name == "execute_bi_query":
            return await self._execute_bi_query(arguments)
        elif name == "get_database_schema":
            return await self._get_database_schema(arguments)
        else:
            raise ValueError(f"Unknown tool: {name}")
    
    async def _execute_bi_query(self, arguments: dict) -> Sequence[TextContent]:
        """Execute a BI query and return results"""
        question = arguments.get("question")
        user_context = arguments.get("user_context", {})
        
        try:
            # Generate SQL
            sql_result = self.sql_agent.generate_sql(question, user_context)
            
            # Execute query
            data = self.db_router.execute_query(
                sql_result["database"],
                sql_result["sql"],
                user_context
            )
            
            # Format response
            response = {
                "sql_generated": sql_result["sql"],
                "explanation": sql_result["explanation"],
                "data": data.to_dict("records"),
                "row_count": len(data),
                "columns": list(data.columns)
            }
            
            return [TextContent(type="text", text=json.dumps(response, indent=2))]
            
        except Exception as e:
            error_response = {"error": str(e), "type": "execution_error"}
            return [TextContent(type="text", text=json.dumps(error_response, indent=2))]
```

---

## Phase 3: Visualization Engine (Weeks 7-8)

### 3.1 Chart Generation Service

```python
# visualization_engine.py
import plotly.graph_objects as go
import plotly.express as px
from typing import Dict, Any
import pandas as pd
import base64
import io

class VisualizationEngine:
    def __init__(self):
        self.chart_types = {
            "bar_chart": self._create_bar_chart,
            "line_chart": self._create_line_chart,
            "pie_chart": self._create_pie_chart,
            "table": self._create_table
        }
    
    def generate_visualization(self, data: pd.DataFrame, viz_type: str, config: Dict = None) -> Dict:
        """Generate visualization from data"""
        if viz_type not in self.chart_types:
            viz_type = self._suggest_visualization_type(data)
        
        chart_func = self.chart_types[viz_type]
        return chart_func(data, config or {})
    
    def _create_bar_chart(self, data: pd.DataFrame, config: Dict) -> Dict:
        """Create bar chart visualization"""
        fig = px.bar(
            data,
            x=config.get('x_column', data.columns[0]),
            y=config.get('y_column', data.columns[1]),
            title=config.get('title', 'Bar Chart'),
            color=config.get('color_column', None)
        )
        
        fig.update_layout(
            template="plotly_white",
            font=dict(family="Arial", size=12)
        )
        
        return {
            "type": "bar_chart",
            "html": fig.to_html(include_plotlyjs="cdn"),
            "json": fig.to_json()
        }
    
    def _create_line_chart(self, data: pd.DataFrame, config: Dict) -> Dict:
        """Create line chart for time series data"""
        fig = px.line(
            data,
            x=config.get('x_column', data.columns[0]),
            y=config.get('y_column', data.columns[1]),
            title=config.get('title', 'Trend Analysis'),
            color=config.get('group_by', None)
        )
        
        return {
            "type": "line_chart",
            "html": fig.to_html(include_plotlyjs="cdn"),
            "json": fig.to_json()
        }
    
    def _suggest_visualization_type(self, data: pd.DataFrame) -> str:
        """Automatically suggest appropriate visualization"""
        if len(data.columns) == 2:
            # Check if first column is date/time
            if pd.api.types.is_datetime64_any_dtype(data.iloc[:, 0]):
                return "line_chart"
            # Check if it's categorical vs numeric
            elif data.iloc[:, 0].dtype == 'object' and pd.api.types.is_numeric_dtype(data.iloc[:, 1]):
                return "bar_chart"
        
        return "table"  # Default fallback
```

---

## Phase 4: Chat Interface Development (Weeks 9-11)

### 4.1 React Chat Component

```javascript
// ChatInterface.jsx
import React, { useState, useEffect, useRef } from 'react';
import { Send, Download, RefreshCw } from 'lucide-react';

const ChatInterface = () => {
  const [messages, setMessages] = useState([]);
  const [inputValue, setInputValue] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const messagesEndRef = useRef(null);

  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
  };

  useEffect(scrollToBottom, [messages]);

  const handleSendMessage = async () => {
    if (!inputValue.trim()) return;

    const userMessage = { type: 'user', content: inputValue, timestamp: new Date() };
    setMessages(prev => [...prev, userMessage]);
    setInputValue('');
    setIsLoading(true);

    try {
      const response = await fetch('/api/query', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ question: inputValue })
      });

      const result = await response.json();
      
      const botMessage = {
        type: 'bot',
        content: result.explanation,
        data: result.data,
        visualization: result.visualization,
        sql: result.sql_generated,
        timestamp: new Date()
      };

      setMessages(prev => [...prev, botMessage]);
    } catch (error) {
      const errorMessage = {
        type: 'bot',
        content: 'Sorry, I encountered an error processing your request.',
        error: true,
        timestamp: new Date()
      };
      setMessages(prev => [...prev, errorMessage]);
    } finally {
      setIsLoading(false);
    }
  };

  const MessageBubble = ({ message }) => (
    <div className={`flex ${message.type === 'user' ? 'justify-end' : 'justify-start'} mb-4`}>
      <div className={`max-w-3xl p-3 rounded-lg ${
        message.type === 'user' 
          ? 'bg-blue-500 text-white' 
          : 'bg-gray-100 text-gray-800'
      }`}>
        <p>{message.content}</p>
        
        {message.data && (
          <div className="mt-3">
            <DataTable data={message.data} />
          </div>
        )}
        
        {message.visualization && (
          <div className="mt-3">
            <div dangerouslySetInnerHTML={{ __html: message.visualization.html }} />
          </div>
        )}
        
        {message.sql && (
          <details className="mt-2">
            <summary className="cursor-pointer text-sm opacity-75">View SQL</summary>
            <pre className="mt-1 p-2 bg-gray-800 text-green-400 text-xs rounded overflow-x-auto">
              {message.sql}
            </pre>
          </details>
        )}
        
        <span className="text-xs opacity-75 block mt-1">
          {message.timestamp.toLocaleTimeString()}
        </span>
      </div>
    </div>
  );

  return (
    <div className="flex flex-col h-screen max-w-6xl mx-auto">
      <header className="bg-blue-600 text-white p-4">
        <h1 className="text-xl font-bold">acme corporation AI Analytics</h1>
        <p className="text-blue-100">Ask any business question in natural language</p>
      </header>
      
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((message, index) => (
          <MessageBubble key={index} message={message} />
        ))}
        {isLoading && (
          <div className="flex justify-start">
            <div className="bg-gray-100 p-3 rounded-lg flex items-center">
              <RefreshCw className="animate-spin mr-2" size={16} />
              Analyzing your question...
            </div>
          </div>
        )}
        <div ref={messagesEndRef} />
      </div>
      
      <div className="border-t p-4">
        <div className="flex space-x-2">
          <input
            type="text"
            value={inputValue}
            onChange={(e) => setInputValue(e.target.value)}
            onKeyPress={(e) => e.key === 'Enter' && handleSendMessage()}
            placeholder="Ask a business question... (e.g., 'Show me revenue by station last quarter')"
            className="flex-1 p-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
            disabled={isLoading}
          />
          <button
            onClick={handleSendMessage}
            disabled={isLoading || !inputValue.trim()}
            className="bg-blue-500 text-white p-2 rounded-lg hover:bg-blue-600 disabled:opacity-50"
          >
            <Send size={20} />
          </button>
        </div>
        
        <div className="mt-2 flex flex-wrap gap-2">
          <SuggestionChip text="Top performing stations this month" onClick={setInputValue} />
          <SuggestionChip text="Revenue trend last 6 months" onClick={setInputValue} />
          <SuggestionChip text="Audience demographics by market" onClick={setInputValue} />
        </div>
      </div>
    </div>
  );
};

export default ChatInterface;
```

---

## Phase 5: Integration & Testing (Weeks 12-14)

### 5.1 API Gateway

```python
# main.py - FastAPI application
from fastapi import FastAPI, HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel
import json
from typing import Optional, Dict

app = FastAPI(title="acme corporation AI Analytics API")
security = HTTPBearer()

class QueryRequest(BaseModel):
    question: str
    context: Optional[Dict] = None

class QueryResponse(BaseModel):
    explanation: str
    data: list
    visualization: Dict
    sql_generated: str
    execution_time: float

@app.post("/api/query", response_model=QueryResponse)
async def execute_query(
    request: QueryRequest, 
    credentials: HTTPAuthorizationCredentials = Depends(security)
):
    """Main endpoint for natural language queries"""
    
    try:
        # Validate user permissions
        user_context = await validate_user_token(credentials.credentials)
        
        # Execute query through MCP
        result = await mcp_client.call_tool(
            "execute_bi_query",
            {
                "question": request.question,
                "user_context": user_context
            }
        )
        
        # Parse MCP response
        query_result = json.loads(result[0].text)
        
        if "error" in query_result:
            raise HTTPException(status_code=400, detail=query_result["error"])
        
        # Generate visualization
        viz_engine = VisualizationEngine()
        data_df = pd.DataFrame(query_result["data"])
        visualization = viz_engine.generate_visualization(data_df, "auto")
        
        return QueryResponse(
            explanation=query_result["explanation"],
            data=query_result["data"],
            visualization=visualization,
            sql_generated=query_result["sql_generated"],
            execution_time=0.0  # TODO: Add timing
        )
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/health")
async def health_check():
    return {"status": "healthy", "version": "1.0.0"}

async def validate_user_token(token: str) -> Dict:
    """Validate JWT token and return user context"""
    # Implement your authentication logic
    return {
        "user_id": "user123",
        "permissions": ["read_revenue", "read_audience"],
        "departments": ["operations", "sales"]
    }
```

### 5.2 Testing Strategy

```python
# tests/test_sql_agent.py
import pytest
from ai_sql_agent import SQLAgent
from database_router import DatabaseRouter

class TestSQLAgent:
    def setup_method(self):
        self.agent = SQLAgent(mock_schemas, mock_llm_client)
    
    def test_revenue_query(self):
        question = "Show me total revenue by station last month"
        result = self.agent.generate_sql(question)
        
        assert "revenue" in result["sql"].lower()
        assert "station" in result["sql"].lower()
        assert result["database"] in ["dwh", "reporting"]
        assert len(result["expected_columns"]) > 0
    
    def test_audience_query(self):
        question = "What are the top 5 demographics for KROQ?"
        result = self.agent.generate_sql(question)
        
        assert "limit 5" in result["sql"].lower()
        assert "kroq" in result["sql"].lower()
        assert result["visualization_type"] in ["bar_chart", "table"]

    def test_security_validation(self):
        malicious_question = "DROP TABLE users; show revenue"
        
        with pytest.raises(SecurityError):
            self.agent.generate_sql(malicious_question)
```

---

## Phase 6: Deployment & Monitoring (Weeks 15-16)

### 6.1 Deployment Configuration

```yaml
# docker-compose.yml
version: '3.8'
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URLS=${DATABASE_URLS}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - redis
      - postgres

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - api

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: ai_bi_logs
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### 6.2 Monitoring and Logging

```python
# monitoring.py
import logging
from prometheus_client import Counter, Histogram, start_http_server
import time
from functools import wraps

# Metrics
query_counter = Counter('ai_bi_queries_total', 'Total BI queries processed')
query_duration = Histogram('ai_bi_query_duration_seconds', 'Query processing time')
error_counter = Counter('ai_bi_errors_total', 'Total errors', ['error_type'])

def monitor_performance(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start_time = time.time()
        query_counter.inc()
        
        try:
            result = func(*args, **kwargs)
            query_duration.observe(time.time() - start_time)
            return result
        except Exception as e:
            error_counter.labels(error_type=type(e).__name__).inc()
            raise
    
    return wrapper

class QueryLogger:
    def __init__(self):
        logging.basicConfig(level=logging.INFO)
        self.logger = logging.getLogger(__name__)
    
    def log_query(self, user_id: str, question: str, sql: str, result_count: int):
        self.logger.info(
            f"Query executed - User: {user_id}, Question: {question[:100]}, "
            f"Rows: {result_count}, SQL: {sql[:200]}..."
        )
```

---

## Implementation Timeline

### Week 1-2: Foundation
- Database schema documentation
- Security requirements definition
- Development environment setup

### Week 3-6: Core Development
- SQL generation agent implementation
- Database router and query executor
- MCP server development
- Basic security implementation

### Week 7-8: Visualization
- Chart generation engine
- Data formatting utilities
- Visualization type auto-detection

### Week 9-11: User Interface
- React chat interface
- API integration
- User experience optimization

### Week 12-14: Testing & Integration
- Unit and integration testing
- Performance optimization
- Security testing
- User acceptance testing

### Week 15-16: Deployment
- Production deployment
- Monitoring setup
- Documentation
- User training

---

## Security Considerations

### Data Access Control
- JWT-based authentication
- Role-based access control (RBAC)
- Row-level security implementation
- Query result filtering based on user permissions

### Query Security
- SQL injection prevention
- Query complexity limits
- Execution timeout controls
- Audit logging for all queries

### Data Privacy
- PII detection and masking
- Data retention policies
- Access logging and monitoring

---

## Success Metrics

### User Experience
- **Response Time:** < 10 seconds for 95% of queries
- **Accuracy:** > 90% of generated SQL queries execute successfully
- **User Adoption:** 80% of dashboard requests converted to chat queries within 6 months

### Technical Performance
- **System Uptime:** 99.5% availability
- **Query Success Rate:** > 95%
- **Concurrent Users:** Support 50+ simultaneous users

### Business Impact
- **Time to Insight:** Reduce from days to minutes
- **Self-Service Adoption:** 70% reduction in manual BI requests
- **Cost Savings:** Reduce dashboard development costs by 60%

---

## Risk Mitigation

### Technical Risks
- **LLM Reliability:** Implement query validation and fallback mechanisms
- **Database Performance:** Query optimization and caching strategies
- **Scalability:** Horizontal scaling architecture design

### Business Risks
- **User Adoption:** Comprehensive training program and gradual rollout
- **Data Accuracy:** Extensive testing with domain experts
- **Compliance:** Regular security audits and data governance reviews

---

## Next Steps

1. **Stakeholder Approval:** Present plan to executive team
2. **Team Assembly:** Identify and assign development resources
3. **Infrastructure Planning:** Provision cloud resources and development environments
4. **Pilot Program:** Start with limited user group and specific use cases
5. **Iterative Development:** Implement in phases with continuous feedback

---

## Appendices

### A. Database Schema Examples
### B. Sample Business Questions and Expected SQL
### C. Security Policy Templates
### D. User Training Materials Outline
### E. Technical Architecture Diagrams

---

**Document Version:** 1.0  
**Last Updated:** August 7, 2025  
**Next Review:** September 1, 2025
