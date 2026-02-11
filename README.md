# ROS2-Week-3-Communication-Patterns
🎯 Learning Objectives
By the end of this week, you will be able to:

✅ Master Topics for pub/sub communication

✅ Implement Services for request/response patterns

✅ Use Actions for long-running tasks with feedback

✅ Understand Quality of Service (QoS) policies

✅ Work with Parameters for runtime configuration

✅ Implement efficient communication between node

3.1 Communication Patterns Overview

ROS 2 offers three main communication patterns:

Topics	

Services

Actions

3.2 Topics - Publisher/Subscriber Pattern

Characteristics:

Asynchronous: Publisher doesn't wait for subscribers

One-to-Many: Multiple nodes can subscribe

Decoupled: Publishers and subscribers don't know each other

Best for: Sensor data, state information, logs

3.3 Services - Request/Response Pattern

Characteristics:

Synchronous: Client waits for response

One-to-One: One server handles requests

Blocking: Client blocks until response

Best for: Commands, calculations, database queries

3.4 Actions - Long-running Tasks

Characteristics:

Asynchronous with feedback

Three-part communication: Goal, Feedback, Result

Cancelable: Can be canceled during execution

Best for: Navigation, manipulation, complex processes

3.5 Quality of Service (QoS)

Why QoS Matters:

Controls how messages are delivered

Balances reliability vs performance

Configurable per topic/service/action

Common QoS Policies:

    from rclpy.qos import QoSProfile, ReliabilityPolicy, DurabilityPolicy, HistoryPolicy

#Reliable delivery (TCP-like)
  
    reliable_qos = QoSProfile(
      depth=10,
      reliability=ReliabilityPolicy.RELIABLE,
      durability=DurabilityPolicy.VOLATILE
    )

#Best effort (UDP-like)

    best_effort_qos = QoSProfile(
      depth=10,
      reliability=ReliabilityPolicy.BEST_EFFORT,
      durability=DurabilityPolicy.VOLATILE  
    )

#Transient local (keep last for late joiners)

    transient_local_qos = QoSProfile(
      depth=10,
      reliability=ReliabilityPolicy.RELIABLE,
      durability=DurabilityPolicy.TRANSIENT_LOCAL
    )
⚙️ Hands-On Setup
Step 1: Create Communication Package

    cd ~/ros2_ws/src
    ros2 pkg create ros2_communication --build-type ament_python \
        --dependencies rclpy std_msgs example_interfaces action_msgs \
        --description "Week 3: ROS 2 Communication Patterns"

    cd ros2_communication
    mkdir -p ros2_communication/{topics,services,actions,parameters}
    
🔧 Practical Exercises

    #!/usr/bin/env python3
    import rclpy
    from rclpy.node import Node
    from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy
    from std_msgs.msg import String
    import time

    class QoSPublisher(Node):
        def __init__(self):
            super().__init__('qos_publisher')
        
        # Create different QoS profiles
            reliable_qos = QoSProfile(
                depth=10,
                reliability=ReliabilityPolicy.RELIABLE,
                history=HistoryPolicy.KEEP_LAST
            )
        
            best_effort_qos = QoSProfile(
                depth=5,
                reliability=ReliabilityPolicy.BEST_EFFORT,
                history=HistoryPolicy.KEEP_LAST
            )
        
        # Create publishers with different QoS
            self.reliable_pub = self.create_publisher(
                String, 'reliable_topic', reliable_qos)
        
            self.best_effort_pub = self.create_publisher(
                String, 'best_effort_topic', best_effort_qos)
        
        # Timer for publishing
            self.timer = self.create_timer(1.0, self.publish_messages)
            self.count = 0
        
            self.get_logger().info("QoS Publisher started")
    
        def publish_messages(self):
            msg = String()
            msg.data = f"Message #{self.count}"
        
        # Publish to reliable topic
            self.reliable_pub.publish(msg)
            self.get_logger().info(f"Published to reliable_topic: {msg.data}")
        
        # Publish to best effort topic (every 2nd message)
            if self.count % 2 == 0:
                self.best_effort_pub.publish(msg)
                self.get_logger().info(f"Published to best_effort_topic: {msg.data}")
        
            self.count += 1

    def main(args=None):
        rclpy.init(args=args)
        node = QoSPublisher()
    
        try:
            rclpy.spin(node)
        except KeyboardInterrupt:
            pass
    
        node.destroy_node()
        rclpy.shutdown()

    if __name__ == '__main__':
        main()

Create topics/subscriber_qos.py:

    #!/usr/bin/env python3
    import rclpy
    from rclpy.node import Node
    from rclpy.qos import QoSProfile, ReliabilityPolicy, qos_profile_sensor_data
    from std_msgs.msg import String

    class QoSSubscriber(Node):
        def __init__(self):
            super().__init__('qos_subscriber')
        
        # Subscriber with reliable QoS
            self.reliable_sub = self.create_subscription(
                String,
                'reliable_topic',
                self.reliable_callback,
                QoSProfile(depth=10, reliability=ReliabilityPolicy.RELIABLE)
            )
        
        # Subscriber with best effort QoS
            self.best_effort_sub = self.create_subscription(
                String,
                'best_effort_topic',
                self.best_effort_callback,
                QoSProfile(depth=5, reliability=ReliabilityPolicy.BEST_EFFORT)
            )
        
        # Subscriber with sensor data QoS (common for sensors)
            self.sensor_sub = self.create_subscription(
                String,
                'sensor_topic',
                self.sensor_callback,
                qos_profile_sensor_data
            )  
        
            self.get_logger().info("QoS Subscriber started")
    
        def reliable_callback(self, msg):
            self.get_logger().info(f"[RELIABLE] Received: {msg.data}")
    
        def best_effort_callback(self, msg):
            self.get_logger().info(f"[BEST_EFFORT] Received: {msg.data}")
    
        def sensor_callback(self, msg):
            self.get_logger().info(f"[SENSOR] Received: {msg.data}")

    def main(args=None):
        rclpy.init(args=args)
        node = QoSSubscriber()
    
        try:
            rclpy.spin(node)
        except KeyboardInterrupt:
            pass
    
        node.destroy_node()
        rclpy.shutdown()

    if __name__ == '__main__':
        main()

  Exercise 2: Services Implementation
  
  Create custom service definition:
  
    cd ~/ros2_ws/src/ros2_communication
    mkdir srv

Create srv/Calculator.srv:
   
    float64 a
    
    float64 b
    ---
    float64 sum
    float64 product
    float64 quotient
    string status

Create services/calculator_server.py:

    #!/usr/bin/env python3
    import rclpy
    from rclpy.node import Node
    from ros2_communication.srv import Calculator
    import time

    class CalculatorServer(Node):
        def __init__(self):
            super().__init__('calculator_server')
        
        # Create service
            self.srv = self.create_service(
                Calculator, 
                'calculate', 
                self.calculate_callback
            )
        
            self.get_logger().info("Calculator Server ready")
    
        def calculate_callback(self, request, response):
            self.get_logger().info(f"Received request: a={request.a}, b={request.b}")
        
        # Simulate some processing time
            time.sleep(1.0)
        
        # Perform calculations
            response.sum = request.a + request.b
            response.product = request.a * request.b
        
        # Handle division by zero
            if request.b != 0:
                response.quotient = request.a / request.b
                response.status = "SUCCESS"
            else:
                response.quotient = 0.0
                response.status = "ERROR: Division by zero"
        
            self.get_logger().info(f"Sending response: {response.status}")
            return response

    def main(args=None):
        rclpy.init(args=args)
        node = CalculatorServer()
    
        try:
            rclpy.spin(node)
        except KeyboardInterrupt:
            pass
    
        node.destroy_node()
        rclpy.shutdown()

    if __name__ == '__main__':
        main()
Create services/calculator_client.py:

    #!/usr/bin/env python3
    import rclpy
    from rclpy.node import Node
    from ros2_communication.srv import Calculator
    import sys

    class CalculatorClient(Node):
        def __init__(self):
            super().__init__('calculator_client')
        
        # Create client
            self.client = self.create_client(Calculator, 'calculate')
        
        # Wait for service to be available
            while not self.client.wait_for_service(timeout_sec=1.0):
                self.get_logger().info('Service not available, waiting...')
        
            self.get_logger().info("Calculator Client ready")
    
    def send_request(self, a, b):
        # Create request
        request = Calculator.Request()
        request.a = float(a)
        request.b = float(b)
        
        # Send request asynchronously
        future = self.client.call_async(request)
        
        # Wait for response
        rclpy.spin_until_future_complete(self, future)
        
        if future.result() is not None:
            response = future.result()
            self.get_logger().info(f"Result:")
            self.get_logger().info(f"  Sum: {response.sum}")
            self.get_logger().info(f"  Product: {response.product}")
            self.get_logger().info(f"  Quotient: {response.quotient}")
            self.get_logger().info(f"  Status: {response.status}")
            return response
        else:
            self.get_logger().error(f"Service call failed: {future.exception()}")
            return None

    def main(args=None):
        rclpy.init(args=args)
    
    # Get numbers from command line or use defaults
        if len(args) > 2:
            a = args[1]
            b = args[2]
        else:
            a = 10.0
            b = 5.0
    
        client = CalculatorClient()
        response = client.send_request(a, b)
    
        client.destroy_node()
        rclpy.shutdown()

    if __name__ == '__main__':
        main()
Exercise 3: Actions Implementation
Create custom action definition:

    cd ~/ros2_ws/src/ros2_communication
    mkdir action
Create action/ProcessImage.action:

#Goal definition

    string image_path
    int32 processing_mode
    ---
#Result definition

    bool success
    string processed_image_path
    int32 processing_time_ms
    ---
#Feedback definition

    float32 progress_percentage
    string current_step

Create actions/image_processor_server.py:

    #!/usr/bin/env python3
    import rclpy
    from rclpy.node import Node
    from rclpy.action import ActionServer
    from ros2_communication.action import ProcessImage
    import time

    class ImageProcessorServer(Node):
        def __init__(self):
            super().__init__('image_processor_server')
        
        # Create action server
            self._action_server = ActionServer(
                self,
                ProcessImage,
                'process_image',
                self.execute_callback
        )
        
        self.get_logger().info("Image Processor Server started")
    
    def execute_callback(self, goal_handle):
        self.get_logger().info(f"Processing image: {goal_handle.request.image_path}")
        
        # Initialize feedback message
        feedback_msg = ProcessImage.Feedback()
        
        # Simulate image processing steps
        steps = [
            ("Loading image", 20),
            ("Pre-processing", 40),
            ("Feature extraction", 60),
            ("Processing", 80),
            ("Saving result", 100)
        ]
        
        for step_name, progress in steps:
            # Check if goal was canceled
            if goal_handle.is_cancel_requested:
                goal_handle.canceled()
                self.get_logger().info('Goal canceled')
                return ProcessImage.Result()
            
            # Update feedback
            feedback_msg.progress_percentage = float(progress)
            feedback_msg.current_step = step_name
            goal_handle.publish_feedback(feedback_msg)
            
            # Simulate work
            time.sleep(1.0)
            self.get_logger().info(f"Progress: {progress}% - {step_name}")
        
        # Mark goal as succeeded
        goal_handle.succeed()
        
        # Create result
        result = ProcessImage.Result()
        result.success = True
        result.processed_image_path = f"processed_{goal_handle.request.image_path}"
        result.processing_time_ms = 5000
        
        self.get_logger().info("Image processing completed successfully")
        return result

    def main(args=None):
        rclpy.init(args=args)
        node = ImageProcessorServer()
    
        try:
            rclpy.spin(node)
        except KeyboardInterrupt:
            pass
    
        node.destroy_node()
        rclpy.shutdown()

    if __name__ == '__main__':
        main()
  Create actions/image_processor_client.py:

    #!/usr/bin/env python3
    import rclpy
    from rclpy.node import Node
    from rclpy.action import ActionClient
    from ros2_communication.action import ProcessImage
    import sys

    class ImageProcessorClient(Node):
        def __init__(self):
            super().__init__('image_processor_client')
        
        # Create action client
            self._action_client = ActionClient(self, ProcessImage, 'process_image')
            self.get_logger().info("Image Processor Client started")
    
        def send_goal(self, image_path, mode=1):
        # Wait for action server
            self.get_logger().info("Waiting for action server...")
            self._action_client.wait_for_server()
        
        # Create goal
            goal_msg = ProcessImage.Goal()
            goal_msg.image_path = image_path
            goal_msg.processing_mode = mode
        
        # Send goal asynchronously
          self.get_logger().info(f"Sending goal: {image_path}")
          future = self._action_client.send_goal_async(
              goal_msg,
              feedback_callback=self.feedback_callback
          )
        
        # Wait for goal to be accepted
          rclpy.spin_until_future_complete(self, future)
          goal_handle = future.result()
        
          if not goal_handle.accepted:
              self.get_logger().error('Goal rejected')
              return
        
          self.get_logger().info('Goal accepted')
        
        # Get result
          result_future = goal_handle.get_result_async()
          rclpy.spin_until_future_complete(self, result_future)
        
        result = result_future.result().result
        self.get_logger().info(f"Result: Success={result.success}")
        self.get_logger().info(f"Processed image: {result.processed_image_path}")
        self.get_logger().info(f"Processing time: {result.processing_time_ms}ms")
        
        return result
    
    def feedback_callback(self, feedback_msg):
        feedback = feedback_msg.feedback
        self.get_logger().info(f"Feedback: {feedback.current_step} - {feedback.progress_percentage:.1f}%")
    
    def cancel_goal(self):
        # Cancel all goals
        future = self._action_client.cancel_all_goals_async()
        rclpy.spin_until_future_complete(self, future)
        self.get_logger().info("All goals cancelled")

    def main(args=None):
        rclpy.init(args=args)
    
    # Get image path from command line or use default
    if len(args) > 1:
        image_path = args[1]
    else:
        image_path = "sample_image.jpg"
    
    client = ImageProcessorClient()
    
    try:
        # Send goal
        result = client.send_goal(image_path)
        
        # You could cancel after some time:
        # time.sleep(2)
        # client.cancel_goal()
        
    except KeyboardInterrupt:
        client.get_logger().info("Cancelling goal...")
        client.cancel_goal()
    
    client.destroy_node()
    rclpy.shutdown()

    if __name__ == '__main__':
        main()
