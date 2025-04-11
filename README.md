# SEP4

## Installation

### Prerequisites

- Docker
- Docker Compose

### Steps

1. **Clone the repository:**

   ```sh
   git clone https://github.com/yourusername/your-repo.git
   cd your-repo
   ```

2. **Create and start containers:**

   ```sh
   docker-compose up -d
   ```

3. **Access the application:**
   - Backend: `http://localhost:8000`
   - Frontend: `http://localhost:3000`
   - Database: `http://localhost:5432`

### Additional Commands

- **Stop containers:**

  ```sh
  docker-compose down
  ```

- **View logs:**

  ```sh
  docker-compose logs
  ```

- **Rebuild containers:**
  ```sh
  docker-compose up --build
  ```

### Notes

- Ensure Docker is running before executing the commands.
- Update the `.env` file with necessary environment variables if required.
- For database migrations, run:
  ```sh
  docker-compose exec backend python manage.py migrate
  ```
