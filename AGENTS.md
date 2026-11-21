## Regole di Sicurezza Tassative
- NON leggere, aprire, stampare, modificare o includere nei commit alcun file `.env` (inclusi `.env.local`, `.env.production`, ecc.).
- Se hai bisogno di conoscere le variabili d'ambiente disponibili, consulta esclusivamente `.env.example`.
- Non eseguire comandi che accedono a percorsi superiori (`../`) rispetto alla directory corrente del progetto.