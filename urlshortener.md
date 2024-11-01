%%{init: {
  'theme': 'neutral',
  'themeVariables': {
    'primaryColor': '#e1e8ef',
    'primaryTextColor': '#2A4252',
    'primaryBorderColor': '#7C9CBF',
    'lineColor': '#2A4252',
    'secondaryColor': '#006100',
    'tertiaryColor': '#fff'
  },
  'flowchart': {
    'curve': 'basis',
    'padding': 20
  }
}}%%
graph TD
    A[Client] -->|API Request| B[Load Balancer]
    B -->|Forward| C[Server Pool]
    C -->|Process| D[Database]
    D -->|Return| C
    C -->|Response| B
    B -->|Final Response| A

    %% Custom styles
    classDef default fill:#e1e8ef,stroke:#7C9CBF,stroke-width:2px;
    classDef highlight fill:#d1fae5,stroke:#059669,stroke-width:2px;
    classDef special fill:#fee2e2,stroke:#dc2626,stroke-width:2px;

    %% Apply styles
    class A,B highlight
    class D special
