Port Optimization API Backend Architecture
*High-Level System Diagram:
```
from flask import Flask, request, jsonify
from flask_redis import FlaskRedis
import pulp
import logging
import time
from multiprocessing import Pool

app = Flask(__name__)
app.config['REDIS_URL'] = 'redis://localhost:6379/0'
redis_store = FlaskRedis(app)

def optimize_port_constraints(data):
    prob = pulp.LpProblem("Port_Optimization", pulp.LpMinimize)
    num_ports = data['num_ports']
    arrival_rates = data['arrival_rates']
    berth_limits = data['berth_limits']
    priority_rules = data['priority_rules']

    x = pulp.LpVariable.dicts("Port", (range(num_ports), range(num_ports)), 0, None, cat='Integer')

    prob += pulp.lpSum([arrival_rates[i] * x[i][j] for i in range(num_ports) for j in range(num_ports)])

    for i in range(num_ports):
        prob += pulp.lpSum([x[i][j] for j in range(num_ports)]) <= berth_limits[i]
    for j in range(num_ports):
        prob += pulp.lpSum([x[i][j] for i in range(num_ports)]) >= priority_rules[j]

    prob.solve()

    solution = []
    for i in range(num_ports):
        for j in range(num_ports):
            if x[i][j].varValue > 0:
                solution.append((i, j, x[i][j].varValue))
    return solution

@app.route('/optimize', methods=['POST'])
def optimize():
    data = request.get_json()
    cache_key = 'optimize:{}'.format(data['num_ports'])
    cached_solution = redis_store.get(cache_key)
    if cached_solution:
        return jsonify({'solution': cached_solution.decode('utf-8')})
    start_time = time.time()
    with Pool(processes=4) as pool:
        solution = pool.apply_async(optimize_port_constraints, args=(data,)).get()
    end_time = time.time()
    logging.info('Optimization took {} seconds'.format(end_time - start_time))
    redis_store.set(cache_key, str(solution))
    return jsonify({'solution': solution})

if __name__ == '__main__':
    app.run(debug=True)

*Example Request:*
```
{
  "num_ports": 3,
  "arrival_rates": [10, 20, 30],
  "berth_limits": [5, 10, 15],
  "priority_rules": [2, 4, 6]
}
```
