# ALX Travel App - Chapa Payment Integration

This project integrates the Chapa API for handling secure payments in the ALX Travel App, allowing users to make bookings with various payment options.

## Features

- **Chapa Payment Integration**: Secure payment processing using Chapa API
- **Payment Status Tracking**: Real-time payment status verification
- **Email Notifications**: Automated email confirmations for successful/failed payments
- **Background Tasks**: Celery integration for async payment processing
- **RESTful API**: Complete API endpoints for payment management

## Prerequisites

- Python 3.8+
- MySQL/MariaDB
- Redis (for Celery)
- Chapa API account

## Installation

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd alx_travel_app_0x02
   ```

2. **Create and activate virtual environment**
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

4. **Set up environment variables**
   Create a `.env` file in the project root:
   ```env
   # Database Configuration
   DB_NAME=alx_travel_app
   DB_USER=your_mysql_username
   DB_PASSWORD=your_mysql_password
   DB_HOST=localhost
   DB_PORT=3306

   # Chapa API Configuration
   CHAPA_SECRET_KEY=your_chapa_secret_key_here
   CHAPA_BASE_URL=https://api.chapa.co/v1

   # Celery Configuration
   CELERY_BROKER_URL=redis://localhost:6379/0
   CELERY_RESULT_BACKEND=redis://localhost:6379/0

   # Email Configuration
   EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
   EMAIL_HOST=localhost
   EMAIL_PORT=1025
   EMAIL_USE_TLS=False
   DEFAULT_FROM_EMAIL=noreply@alxtravel.com
   ```

5. **Set up database**
   ```bash
   python manage.py makemigrations
   python manage.py migrate
   ```

6. **Create superuser**
   ```bash
   python manage.py createsuperuser
   ```

## Chapa API Setup

1. **Create Chapa Account**
   - Visit [Chapa Developer Portal](https://developer.chapa.co/)
   - Sign up for an account
   - Navigate to the dashboard

2. **Get API Keys**
   - Go to API Keys section
   - Copy your Secret Key
   - Update the `CHAPA_SECRET_KEY` in your `.env` file

3. **Test with Sandbox**
   - Use Chapa's sandbox environment for testing
   - Test payment flows before going live

## API Endpoints

### Payment Endpoints

#### 1. Create Payment
```http
POST /api/payments/
Content-Type: application/json
Authorization: Bearer <token>

{
    "booking": 1,
    "amount": "5000.00",
    "currency": "ETB",
    "payment_method": "card"
}
```

#### 2. Initiate Payment with Chapa
```http
POST /api/payments/{payment_id}/initiate_payment/
Content-Type: application/json
Authorization: Bearer <token>

{
    "payment_method": "card",
    "customer_phone": "+251912345678"
}
```

**Response:**
```json
{
    "success": true,
    "checkout_url": "https://checkout.chapa.co/...",
    "payment_id": 1,
    "reference": "payment-ref-123",
    "message": "Payment initiated successfully. Please complete payment using the checkout URL."
}
```

#### 3. Verify Payment Status
```http
POST /api/payments/{payment_id}/verify_payment/
Authorization: Bearer <token>
```

**Response:**
```json
{
    "success": true,
    "status": "completed",
    "amount": "5000.00",
    "currency": "ETB",
    "message": "Payment status: completed"
}
```

#### 4. Get Payment Status
```http
GET /api/payments/{payment_id}/payment_status/
Authorization: Bearer <token>
```

#### 5. List User Payments
```http
GET /api/payments/my_payments/
Authorization: Bearer <token>
```

### Booking Endpoints

#### 1. Create Booking (Automatically creates payment)
```http
POST /api/bookings/
Content-Type: application/json
Authorization: Bearer <token>

{
    "listing": 1,
    "start_date": "2024-01-15",
    "end_date": "2024-01-18"
}
```

#### 2. Get Booking Payment Details
```http
GET /api/bookings/{booking_id}/payment_details/
Authorization: Bearer <token>
```

## Payment Workflow

1. **User creates a booking**
   - System automatically creates a payment record
   - Payment status is set to "pending"

2. **User initiates payment**
   - Call the initiate_payment endpoint
   - System creates Chapa transaction
   - Returns checkout URL to user

3. **User completes payment**
   - User is redirected to Chapa checkout
   - Completes payment using preferred method

4. **Payment verification**
   - System verifies payment with Chapa API
   - Updates payment status accordingly
   - Sends confirmation email if successful

5. **Background processing**
   - Celery tasks handle email notifications
   - Payment status updates are processed asynchronously

## Payment Statuses

- **pending**: Payment created but not initiated
- **processing**: Payment initiated with Chapa
- **completed**: Payment successfully processed
- **failed**: Payment failed or expired
- **cancelled**: Payment was cancelled

## Email Templates

The system includes email templates for:
- Payment confirmation
- Payment failure notification
- Booking confirmation

Templates are located in `listings/templates/listings/email/`

## Background Tasks

### Celery Tasks

1. **send_payment_confirmation_email**: Sends confirmation email for successful payments
2. **send_payment_failed_email**: Sends notification for failed payments
3. **update_payment_status**: Verifies payment status with Chapa API
4. **cleanup_expired_payments**: Cleans up payments pending for more than 24 hours

### Running Celery

```bash
# Start Celery worker
celery -A alx_travel_app worker -l info

# Start Celery beat (for scheduled tasks)
celery -A alx_travel_app beat -l info
```

## Testing

### Test Payment Flow

1. **Create a test booking**
   ```bash
   curl -X POST http://localhost:8000/api/bookings/ \
     -H "Authorization: Bearer <token>" \
     -H "Content-Type: application/json" \
     -d '{"listing": 1, "start_date": "2024-01-15", "end_date": "2024-01-18"}'
   ```

2. **Initiate payment**
   ```bash
   curl -X POST http://localhost:8000/api/payments/1/initiate_payment/ \
     -H "Authorization: Bearer <token>" \
     -H "Content-Type: application/json" \
     -d '{"payment_method": "card"}'
   ```

3. **Verify payment status**
   ```bash
   curl -X POST http://localhost:8000/api/payments/1/verify_payment/ \
     -H "Authorization: Bearer <token>"
   ```

### Using Chapa Sandbox

For testing, use Chapa's sandbox environment:
- Test card numbers are provided in Chapa documentation
- No real money is charged
- Perfect for development and testing

## Error Handling

The system includes comprehensive error handling for:
- API connection failures
- Invalid payment data
- Network timeouts
- Chapa API errors

All errors are logged and appropriate responses are returned to the client.

## Security Considerations

1. **API Key Security**: Never commit API keys to version control
2. **HTTPS**: Always use HTTPS in production
3. **Input Validation**: All inputs are validated before processing
4. **Rate Limiting**: Consider implementing rate limiting for payment endpoints
5. **Logging**: Sensitive data is not logged

## Production Deployment

1. **Update environment variables**
   - Use production Chapa API keys
   - Configure production database
   - Set up proper email backend

2. **Security settings**
   - Set `DEBUG=False`
   - Configure `ALLOWED_HOSTS`
   - Use HTTPS

3. **Monitoring**
   - Monitor payment success rates
   - Set up error alerting
   - Monitor Celery task performance

## Troubleshooting

### Common Issues

1. **Database Connection Error**
   - Check database credentials in `.env`
   - Ensure MySQL service is running

2. **Chapa API Errors**
   - Verify API key is correct
   - Check API endpoint URLs
   - Ensure account is activated

3. **Celery Issues**
   - Check Redis connection
   - Verify Celery worker is running
   - Check task logs

### Logs

Check logs in:
- Django logs: `django.log`
- Celery logs: Console output
- Application logs: Check logging configuration

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests
5. Submit a pull request

## License

This project is licensed under the MIT License.

## Support

For support, please contact the development team or create an issue in the repository.
