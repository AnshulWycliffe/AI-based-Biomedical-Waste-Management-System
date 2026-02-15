# Implementation Plan: AWS Anomaly Detection

## Overview

This implementation plan breaks down the AWS anomaly detection feature into discrete coding tasks. The approach follows a modular design where each component is built and tested incrementally, starting with the core Lambda function, then Flask integration, and finally the civic dashboard.

The implementation prioritizes the critical path (waste creation must never fail) and includes property-based tests alongside unit tests for comprehensive validation.

## Tasks

- [ ] 1. Set up AWS Lambda function infrastructure
  - Create Lambda function deployment package structure
  - Set up Python 3.11 runtime configuration
  - Configure CloudWatch logging permissions
  - Create IAM role with necessary permissions (CloudWatch Logs write)
  - _Requirements: 3.1, 4.1-4.5_

- [ ] 2. Implement core anomaly detection logic
  - [ ] 2.1 Create Lambda handler function with input validation
    - Implement lambda_handler function
    - Parse and validate input JSON (facility_id, current_quantity, historical_quantities)
    - Return error response for missing or invalid fields
    - _Requirements: 3.5, 2.3_
  
  - [ ] 2.2 Implement statistical calculations
    - Calculate mean using statistics.mean()
    - Calculate standard deviation using statistics.stdev()
    - Compute Z-score: (current_quantity - mean) / std_dev
    - Handle edge cases: std_dev=0, len(historical_quantities)<5
    - _Requirements: 1.1, 1.2, 1.3, 2.1, 2.2_
  
  - [ ] 2.3 Implement anomaly flagging logic
    - Compare absolute Z-score against threshold (2.5)
    - Set is_anomaly flag based on threshold
    - Format output JSON with all required fields
    - _Requirements: 1.4, 1.5, 1.6_
  
  - [ ] 2.4 Add CloudWatch logging
    - Log facility_id, current_quantity, z_score, anomaly_detected, timestamp
    - Use structured JSON logging format
    - Ensure ISO 8601 timestamp format
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_
  
  - [ ]* 2.5 Write property test for mean calculation
    - **Property 1: Mean calculation correctness**
    - **Validates: Requirements 1.1**
  
  - [ ]* 2.6 Write property test for standard deviation calculation
    - **Property 2: Standard deviation calculation correctness**
    - **Validates: Requirements 1.2**
  
  - [ ]* 2.7 Write property test for Z-score formula
    - **Property 3: Z-score formula correctness**
    - **Validates: Requirements 1.3**
  
  - [ ]* 2.8 Write property test for anomaly threshold
    - **Property 4: Anomaly threshold detection**
    - **Validates: Requirements 1.4, 1.5**
  
  - [ ]* 2.9 Write property test for output structure
    - **Property 5: Output structure completeness**
    - **Validates: Requirements 1.6, 3.6**
  
  - [ ]* 2.10 Write unit tests for edge cases
    - Test zero standard deviation case
    - Test insufficient historical data (<5 entries)
    - Test empty historical data
    - _Requirements: 2.1, 2.2_

- [ ] 3. Create MongoDB data model
  - [ ] 3.1 Implement WasteAnomaly MongoEngine document
    - Define all fields: facility_id, waste_id, actual_quantity, mean, std_dev, z_score, flagged, created_at
    - Set default values (flagged=True, created_at=now in IST)
    - Add indexes for facility_id, created_at, flagged
    - _Requirements: 6.1-6.8_
  
  - [ ]* 3.2 Write property test for WasteAnomaly record structure
    - **Property 15: WasteAnomaly record structure completeness**
    - **Validates: Requirements 6.1-6.8**

- [ ] 4. Implement Flask anomaly detection service
  - [ ] 4.1 Create AnomalyDetectionService class
    - Initialize with boto3 Lambda client and function name
    - Set timeout configuration (5 seconds)
    - _Requirements: 5.7_
  
  - [ ] 4.2 Implement historical data retrieval method
    - Query Waste collection by facility_id
    - Filter for last 30 days using IST timezone
    - Extract only quantity field
    - Sort by created_at descending
    - Handle empty results (return empty list)
    - _Requirements: 5.2, 9.1, 9.2, 9.3, 9.4, 9.5, 12.4_
  
  - [ ] 4.3 Implement Lambda invocation method
    - Build payload with facility_id, current_quantity, historical_quantities
    - Invoke Lambda using boto3 with RequestResponse invocation type
    - Parse response JSON
    - Handle all error cases: connection failures, timeouts, malformed responses, error status codes
    - Return None on any error (never raise exceptions)
    - _Requirements: 5.3, 5.4, 8.1, 8.2, 8.3_
  
  - [ ] 4.4 Implement anomaly record save method
    - Create WasteAnomaly document with all fields
    - Set created_at to current time in IST
    - Save to MongoDB
    - Handle save failures gracefully (log and return False)
    - _Requirements: 5.5, 12.1_
  
  - [ ]* 4.5 Write property test for historical data retrieval
    - **Property 10: Historical data retrieval correctness**
    - **Validates: Requirements 5.2, 9.1, 9.2, 9.3, 9.4**
  
  - [ ]* 4.6 Write property test for Lambda invocation parameters
    - **Property 11: Lambda invocation with correct parameters**
    - **Validates: Requirements 5.3**
  
  - [ ]* 4.7 Write property test for response parsing
    - **Property 12: Response parsing correctness**
    - **Validates: Requirements 5.4**
  
  - [ ]* 4.8 Write property test for conditional anomaly record creation
    - **Property 13: Conditional anomaly record creation**
    - **Validates: Requirements 5.5**
  
  - [ ]* 4.9 Write property test for error resilience
    - **Property 14: Error resilience for waste creation**
    - **Validates: Requirements 5.6, 8.1, 8.2, 8.3, 8.4, 8.5**

- [ ] 5. Checkpoint - Ensure core anomaly detection works
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Integrate anomaly detection into waste creation endpoint
  - [ ] 6.1 Modify POST /api/waste route
    - Save Waste entry to MongoDB first (primary operation)
    - Call anomaly service to get historical quantities
    - Call anomaly service to check for anomaly
    - If is_anomaly=true, save WasteAnomaly record
    - Wrap anomaly detection in try-except to ensure waste creation always succeeds
    - Return success response regardless of anomaly detection outcome
    - _Requirements: 5.1, 5.6_
  
  - [ ]* 6.2 Write property test for waste persistence before detection
    - **Property 9: Waste entry persistence before detection**
    - **Validates: Requirements 5.1**
  
  - [ ]* 6.3 Write integration test for complete waste creation flow
    - Test normal case (no anomaly)
    - Test anomaly case (Z-score >= 2.5)
    - Test Lambda failure case (waste still created)
    - _Requirements: 5.1-5.6_

- [ ] 7. Implement civic dashboard endpoint
  - [ ] 7.1 Create GET /api/civic/dashboard route
    - Add @login_required decorator
    - Verify user role equals "civic" (return 403 if not)
    - Calculate today's date boundaries in IST timezone
    - Query count of anomalies created today
    - Query 10 most recent WasteAnomaly records
    - Identify high-risk facilities (3+ anomalies in last 7 days)
    - Format response with all required fields
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 11.1, 11.2_
  
  - [ ]* 7.2 Write property test for today's anomaly count
    - **Property 16: Today's anomaly count accuracy**
    - **Validates: Requirements 7.1, 7.2, 12.3**
  
  - [ ]* 7.3 Write property test for recent anomalies retrieval
    - **Property 17: Recent anomalies retrieval**
    - **Validates: Requirements 7.3, 7.4**
  
  - [ ]* 7.4 Write property test for high-risk facility identification
    - **Property 18: High-risk facility identification**
    - **Validates: Requirements 7.5**
  
  - [ ]* 7.5 Write property test for role-based authorization
    - **Property 19: Role-based authorization**
    - **Validates: Requirements 11.1, 11.2**
  
  - [ ]* 7.6 Write unit tests for dashboard edge cases
    - Test with no anomalies
    - Test with exactly 10 anomalies
    - Test with more than 10 anomalies
    - Test unauthorized access (non-civic user)
    - _Requirements: 11.1, 11.2_

- [ ] 8. Implement facility data privacy
  - [ ] 8.1 Modify facility waste entry view endpoint
    - Ensure response does not include anomaly flags or Z-scores
    - Filter out WasteAnomaly data from facility user responses
    - _Requirements: 11.3_
  
  - [ ]* 8.2 Write property test for facility data privacy
    - **Property 20: Facility data privacy**
    - **Validates: Requirements 11.3**

- [ ] 9. Implement civic cross-facility visibility
  - [ ] 9.1 Create GET /api/civic/anomalies endpoint
    - Verify user role equals "civic"
    - Query all WasteAnomaly records without facility_id filter
    - Support pagination (optional)
    - Return all anomalies across all facilities
    - _Requirements: 11.4_
  
  - [ ]* 9.2 Write property test for civic cross-facility visibility
    - **Property 21: Civic cross-facility visibility**
    - **Validates: Requirements 11.4**

- [ ] 10. Implement timezone consistency
  - [ ]* 10.1 Write property test for IST timestamp consistency
    - **Property 22: IST timestamp consistency**
    - **Validates: Requirements 12.1, 12.2**
  
  - [ ]* 10.2 Write property test for IST date range calculations
    - **Property 23: IST date range calculations**
    - **Validates: Requirements 12.4**
  
  - [ ]* 10.3 Write unit tests for timezone edge cases
    - Test date boundary calculations (midnight IST)
    - Test daylight saving time handling (if applicable)
    - _Requirements: 12.1, 12.2, 12.3, 12.4_

- [ ] 11. Checkpoint - Ensure all core features work
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 12. (Optional) Implement S3 report storage
  - [ ] 12.1 Add S3 configuration to Flask app
    - Add S3 bucket name to configuration
    - Initialize boto3 S3 client
    - Add feature flag for S3 integration
    - _Requirements: 10.1_
  
  - [ ] 12.2 Implement report generation method
    - Generate JSON report with anomaly details
    - Include facility_id, waste_id, timestamp, statistics, Z-score
    - Format report for readability
    - _Requirements: 10.1_
  
  - [ ] 12.3 Implement S3 upload method
    - Upload report to S3 bucket with key: {facility_id}/{timestamp}.json
    - Store S3 object key in WasteAnomaly record
    - Handle upload failures gracefully (log and continue)
    - _Requirements: 10.2, 10.3, 10.5_
  
  - [ ] 12.4 Add S3 download link to dashboard response
    - Generate presigned URL for S3 object
    - Include download link in anomaly record response
    - _Requirements: 10.4_
  
  - [ ]* 12.5 Write property test for S3 report generation and upload
    - **Property 24: S3 report generation and upload**
    - **Validates: Requirements 10.1, 10.2, 10.3**
  
  - [ ]* 12.6 Write property test for S3 download link provision
    - **Property 25: S3 download link provision**
    - **Validates: Requirements 10.4**
  
  - [ ]* 12.7 Write property test for S3 error resilience
    - **Property 26: S3 error resilience**
    - **Validates: Requirements 10.5**

- [ ] 13. Deploy Lambda function to AWS
  - Package Lambda function code
  - Create deployment ZIP with dependencies
  - Deploy to AWS Lambda in ap-south-1 region
  - Configure environment variables
  - Test Lambda function in AWS console
  - _Requirements: 3.1_

- [ ] 14. Configure Flask application for production
  - Add AWS credentials configuration (IAM role or environment variables)
  - Set Lambda function name in configuration
  - Set S3 bucket name (if using optional feature)
  - Configure logging levels
  - _Requirements: 5.3_

- [ ] 15. Final integration testing
  - [ ]* 15.1 Run all property tests (100 iterations each)
    - Verify all 26 properties pass
  
  - [ ]* 15.2 Run all unit tests
    - Verify edge cases and error conditions
  
  - [ ]* 15.3 Run integration tests
    - Test complete waste creation flow
    - Test civic dashboard flow
    - Test error scenarios
  
  - [ ] 15.4 Manual testing checklist
    - Create waste entry as facility user (verify success)
    - Create anomalous waste entry (verify anomaly detected)
    - View civic dashboard (verify KPIs and recent anomalies)
    - Test Lambda failure scenario (verify waste still created)
    - Test unauthorized access (verify 403 error)

- [ ] 16. Final checkpoint - Production readiness
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties (minimum 100 iterations each)
- Unit tests validate specific examples and edge cases
- The critical path (waste creation) is prioritized and must never fail
- S3 integration (tasks 12.x) is entirely optional and can be implemented later
- Use `hypothesis` library for property-based testing in Python
- Use `moto` library for mocking AWS services in tests
- Use `freezegun` library for time-based testing (timezone tests)
