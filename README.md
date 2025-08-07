# نکات پیشرفته Celery  برای استفاده در پروژه های مالی django 
## About
توسعه پروژه های مالی مانند سامانه های پرداخت، صرافی آنلاین و خرید و فروش آنلاین طلا نیازمند بکارگیری بهترین استانداردهای امنیتی به منظور تضمین محافظت از داده ها، پایداری و عملکرد درست سامانه، consistency و integrity داده ها می باشد. یکی از مهم ترین اجزا هر پروژه django بخش Async Task Queue می باشد که شاخص ترین آن Celery می باشد.
در این ریپو تلاش کرده ام مجموعه ای از بهترین نکات سلری که می بایست در پروژه های مالی پیاده سازه شوند جمع آوری کنم.

## Celery Topics
1- **Basic Celery Configuration**
```
# settings.py
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_TIME_LIMIT = 300  # 5 minutes max for banking tasks
CELERY_TASK_SOFT_TIME_LIMIT = 240  # Soft limit before hard limit
CELERY_WORKER_PREFETCH_MULTIPLIER = 1  # For financial accuracy
CELERY_TASK_ACKS_LATE = True  # Ensure task completion
CELERY_WORKER_DISABLE_RATE_LIMITS = False
CELERY_TASK_REJECT_ON_WORKER_LOST = True
```

2- **Handling task failures and retries in critical banking operations**
```
from celery import Task
from celery.exceptions import Retry

class BankingTask(Task):
    autoretry_for = (ConnectionError, TimeoutError)
    retry_kwargs = {'max_retries': 3, 'countdown': 60}
    retry_backoff = True
    retry_jitter = False  # Deterministic for banking
    
@celery_app.task(bind=True, base=BankingTask)
def process_payment(self, payment_id):
    try:
        payment = Payment.objects.get(id=payment_id)
        # Process payment logic
        result = external_payment_api.process(payment)
        
        # Log for audit
        AuditLog.objects.create(
            task_id=self.request.id,
            action='payment_processed',
            payment_id=payment_id,
            status='success'
        )
        return result
        
    except TemporaryFailure as exc:
        # Log retry attempt
        AuditLog.objects.create(
            task_id=self.request.id,
            action='payment_retry',
            payment_id=payment_id,
            attempt=self.request.retries + 1
        )
        raise self.retry(exc=exc, countdown=60)
        
    except PermanentFailure:
        # Mark payment as failed, notify customer
        payment.status = 'failed'
        payment.save()
        send_failure_notification.delay(payment_id)
        raise
```

3- **Task routing for different banking operations**
```
# settings.py
CELERY_TASK_ROUTES = {
    'banking.tasks.process_wire_transfer': {'queue': 'critical'},
    'banking.tasks.process_ach_payment': {'queue': 'standard'},
    'banking.tasks.generate_statement': {'queue': 'reports'},
    'banking.tasks.fraud_check': {'queue': 'security'},
}

CELERY_TASK_ANNOTATIONS = {
    'banking.tasks.process_wire_transfer': {'rate_limit': '10/s'},
    'banking.tasks.generate_statement': {'rate_limit': '2/m'},
}
```

4- **Celery workflows for complex banking processes**
- **Groups**: Parallel execution
- **Chains**: Sequential execution
- **Chords**: Parallel then callback
- **Map/Starmap**: Apply task to multiple arguments
```
from celery import group, chain, chord

# Complex loan approval workflow
def process_loan_application(application_id):
    # Parallel checks
    background_checks = group(
        credit_check.s(application_id),
        employment_verification.s(application_id),
        fraud_check.s(application_id),
        document_verification.s(application_id)
    )
    
    # Sequential processing after checks
    # The callback gets list of results from group as its first argument
    approval_chain = chain(
        compile_check_results.s(),                # input: list of check results
        calculate_risk_score.s(application_id),   # input: compiled results + application_id
        make_approval_decision.s(application_id), # input: risk score + application_id
        update_application_status.s(application_id)  # final update
    )
    
    # Chord: run checks in parallel, then process results
    workflow = chord(background_checks)(approval_chain)
    return workflow.apply_async()

# Usage
result = process_loan_application(12345)
```

5- **security considerations for Celery in banking**
```
# Message Encryption
CELERY_TASK_SERIALIZER = 'json'
CELERY_ACCEPT_CONTENT = ['json']
BROKER_USE_SSL = {
    'keyfile': '/path/to/key.pem',
    'certfile': '/path/to/cert.pem',
    'ca_certs': '/path/to/ca.pem',
    'cert_reqs': ssl.CERT_REQUIRED,
}
# Task Signing
CELERY_TASK_ALWAYS_EAGER = False
CELERY_TASK_STORE_EAGER_RESULT = True
CELERY_SECURITY_KEY = 'your-secret-key'
CELERY_SECURITY_CERTIFICATE = '/path/to/cert.pem'
CELERY_SECURITY_CERT_STORE = '/path/to/certs'
```

6- **Custom task classes for banking operations with Retry, Failure & Success Mechanisms**
```
from celery import Task
from django.core.mail import send_mail
from .models import AuditLog, TaskExecutionLog

class BankingBaseTask(Task):
    """Base task class for all banking operations"""
    
    abstract = True
    max_retries = 3
    default_retry_delay = 60
    
    def on_retry(self, exc, task_id, args, kwargs, einfo):
        """Log retry attempts for compliance"""
        AuditLog.objects.create(
            task_id=task_id,
            task_name=self.name,
            action='retry',
            retry_count=self.request.retries,
            exception=str(exc),
            timestamp=timezone.now()
        )
    
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        """Handle task failures with proper logging and alerting"""
        AuditLog.objects.create(
            task_id=task_id,
            task_name=self.name,
            action='failure',
            exception=str(exc),
            traceback=einfo.traceback,
            timestamp=timezone.now()
        )
        
        # Alert operations team for critical tasks
        if self.name.startswith('banking.critical'):
            send_mail(
                subject=f'Critical Banking Task Failed: {self.name}',
                message=f'Task {task_id} failed with error: {exc}',
                from_email='alerts@bank.com',
                recipient_list=['ops-team@bank.com']
            )
    
    def on_success(self, retval, task_id, args, kwargs):
        """Log successful task completion"""
        TaskExecutionLog.objects.create(
            task_id=task_id,
            task_name=self.name,
            status='success',
            result=str(retval)[:1000],  # Truncate large results
            execution_time=self.request.timelimit,
            timestamp=timezone.now()
        )

class CriticalBankingTask(BankingBaseTask):
    """For critical operations like wire transfers"""
    max_retries = 1  # Fewer retries for critical tasks
    time_limit = 300  # 5 minute timeout
    soft_time_limit = 240

@celery_app.task(bind=True, base=CriticalBankingTask)
def process_wire_transfer(self, transfer_id):
    # Implementation here
    pass
```

7- **Implement task callbacks and error handling for payment workflows**
```
from celery import chain, group
from celery.exceptions import Retry

@celery_app.task(bind=True)
def process_payment(self, payment_id):
    try:
        payment = Payment.objects.get(id=payment_id)
        # Payment processing logic
        result = external_payment_service.charge(payment)
        
        # Success callback
        payment_success_callback.delay(payment_id, result)
        return result
        
    except PaymentDeclined as exc:
        # Handle declined payment
        payment_declined_callback.delay(payment_id, str(exc))
        raise
        
    except NetworkError as exc:
        # Retry for network issues
        if self.request.retries < self.max_retries:
            raise self.retry(exc=exc, countdown=60)
        else:
            payment_failure_callback.delay(payment_id, 'network_timeout')
            raise

@celery_app.task
def payment_success_callback(payment_id, result):
    """Handle successful payment"""
    payment = Payment.objects.get(id=payment_id)
    payment.status = 'completed'
    payment.external_reference = result.get('transaction_id')
    payment.completed_at = timezone.now()
    payment.save()
    
    # Send confirmation
    send_payment_confirmation.delay(payment.customer.email, payment_id)
    
    # Update accounting
    update_accounting_records.delay(payment_id)

@celery_app.task
def payment_declined_callback(payment_id, reason):
    """Handle declined payment"""
    payment = Payment.objects.get(id=payment_id)
    payment.status = 'declined'
    payment.decline_reason = reason
    payment.save()
    
    # Notify customer
    send_payment_declined_notification.delay(payment.customer.email, payment_id)

@celery_app.task
def payment_failure_callback(payment_id, error_type):
    """Handle payment processing failure"""
    payment = Payment.objects.get(id=payment_id)
    payment.status = 'failed'
    payment.failure_reason = error_type
    payment.save()
    
    # Alert operations team
    alert_operations_team.delay(f'Payment {payment_id} failed: {error_type}')

# Usage with error handling
def initiate_payment_workflow(payment_id):
    # Chain with error callbacks
    workflow = chain(
        process_payment.s(payment_id),
        # Success path continues here
        finalize_payment.s(payment_id)
    ).apply_async()
    
    return workflow
```
