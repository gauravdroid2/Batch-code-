# Batch-code-
#Make a .bat file with code that can run a job or transformation(.kjb or .ktr) without opening pentaho
#after making the .bat file, use it in windows task scheduler
@echo off
setlocal enabledelayedexpansion

REM ==== Configuration ====
set CARTE_HOST=localhost
set CARTE_PORT=8080
set JOB_FILE=C:\Users\diwagar.s-ext\Downloads\Pentaho jobs\Hana_replication.kjb
set PDI_HOME=C:\Users\diwagar.s-ext\Downloads\pdi-ce-10.2.0.0-222\data-integration
set LOG_DIR=C:\Users\diwagar.s-ext\Downloads\pdi-ce-10.2.0.0-222\data-integration\pentaho files\logs

REM ==== Create log folder if not exists ====
if not exist "%LOG_DIR%" mkdir "%LOG_DIR%"

REM ==== Timestamp for log file ====
for /f "tokens=1-4 delims=/ " %%a in ('date /t') do (
  set mydate=%%d-%%b-%%c
)
for /f "tokens=1-2 delims=: " %%a in ('time /t') do (
  set mytime=%%a%%b
)
set LOG_FILE_JOB=%LOG_DIR%\job_!mydate!_!mytime!.log
set LOG_FILE_CARTE=%LOG_DIR%\carte_!mydate!_!mytime!.log

echo ==== Starting Pentaho job at %date% %time% ==== > "%LOG_FILE_JOB%"

REM ==== Step 1: Start Carte in background with its own log ====
start "" /B cmd /c "%PDI_HOME%\carte.bat %CARTE_HOST% %CARTE_PORT% >> "%LOG_FILE_CARTE%" 2>&1"

REM ==== Step 2: Wait for Carte to fully start ====
timeout /t 10 >nul

REM ==== Step 3: Run the job via Kitchen with summary logs ====
echo Running job %JOB_FILE% >> "%LOG_FILE_JOB%"
"%PDI_HOME%\kitchen.bat" /file:"%JOB_FILE%" /level:Basic >> "%LOG_FILE_JOB%" 2>&1

REM ==== Step 4: Stop Carte after job finishes ====
echo Stopping Carte server... >> "%LOG_FILE_JOB%"
curl -s -u cluster:cluster "http://%CARTE_HOST%:%CARTE_PORT%/kettle/stopCarte" >> "%LOG_FILE_JOB%" 2>&1

REM ==== Step 5: Final status ====
findstr /C:"ERROR" "%LOG_FILE_JOB%" >nul
if %errorlevel%==0 (
    echo ==== Job Completed WITH ERRORS at %date% %time% ==== >> "%LOG_FILE_JOB%"
) else (
    echo ==== Job Completed SUCCESSFULLY at %date% %time% ==== >> "%LOG_FILE_JOB%"
)

endlocal
exit
