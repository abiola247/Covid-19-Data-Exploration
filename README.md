# COVID-19 Data Exploration Using SQL

Exploratory SQL analysis of global COVID-19 data, examining death rates, infection trends, and vaccination progress across countries and continents.

## Dataset
Two source files:
- **CovidDeaths** — global COVID-19 cases, deaths, and population data by location and date
- **CovidVaccinations** — vaccination data by location and date

## Tools Used
- **SQL** — the main language used for exploration
- **Microsoft Excel** — source format of the raw data

## Questions Answered
1. What is the percentage death rate across different locations and continents?
2. What percentage of the population has been infected with COVID-19?
3. Which countries have the highest infection rates relative to their populations?
4. Which continents have the highest death counts?
5. How many people in total have been vaccinated against COVID-19?
6. What is the percentage of the population vaccinated in each location?

## The Analysis

### 1. Death rate by location and continent
```sql
SELECT location,
       SUM(total_cases) AS total_cases,
       SUM(total_deaths) AS total_deaths,
       (SUM(total_deaths) * 1.0 / SUM(total_cases)) * 100 AS TotalDeathPercentage
FROM CovidDeaths
WHERE total_deaths IS NOT NULL
GROUP BY location;
```

### 2. Infection rate relative to population
```sql
SELECT location, date, total_cases, population,
       ROUND(((total_cases * 1.0 / population) * 100), 5) AS PopulationPercentage
FROM CovidDeaths
ORDER BY 1, 2;
```

### 3. Countries with the highest infection rates
```sql
SELECT location, population,
       MAX(total_cases) AS HighestInfectionCount,
       MAX(ROUND(((total_cases * 1.0 / population) * 100), 5)) AS PopulationPercentage
FROM CovidDeaths
GROUP BY population, location
ORDER BY PopulationPercentage DESC;
```

### 4. Continents with the highest death counts
```sql
SELECT continent, MAX(CAST(total_deaths AS INT)) AS TotalDeathCount
FROM CovidDeaths
WHERE continent IS NOT NULL
GROUP BY continent
ORDER BY TotalDeathCount DESC;
```

### 5. Rolling vaccination totals with a window function
```sql
SELECT dea.continent, dea.location, dea.date, dea.population, vac.new_vaccinations,
       SUM(CAST(vac.new_vaccinations AS INT))
         OVER (PARTITION BY dea.location ORDER BY dea.location, dea.date) AS RollingPeopleVaccinated
FROM CovidDeaths dea
JOIN CovidVaccinations vac
  ON dea.location = vac.location
 AND dea.date = vac.date
WHERE dea.continent IS NOT NULL
ORDER BY 2, 3;
```

### 6. Vaccinated population percentage with a CTE
```sql
WITH PopvsVac (continent, location, date, population, new_vaccinations, RollingPeopleVaccinated)
AS (
  SELECT dea.continent, dea.location, dea.date, dea.population, vac.new_vaccinations,
         SUM(CAST(vac.new_vaccinations AS INT))
           OVER (PARTITION BY dea.location ORDER BY dea.location, dea.date) AS RollingPeopleVaccinated
  FROM CovidDeaths dea
  JOIN CovidVaccinations vac
    ON dea.location = vac.location
   AND dea.date = vac.date
  WHERE dea.continent IS NOT NULL
)
SELECT *, (RollingPeopleVaccinated / Population) * 100 AS PeopleVaccinatedPercentage
FROM PopvsVac;
```

### 7. Storing results for reuse — temp table and view
```sql
DROP TABLE IF EXISTS PercentPopulationVaccinated;

CREATE TEMP TABLE PercentPopulationVaccinated (
  Continent TEXT,
  Location TEXT,
  Date TEXT,
  Population INTEGER,
  New_vaccinations INTEGER,
  RollingPeopleVaccinated INTEGER
);

INSERT INTO PercentPopulationVaccinated
SELECT dea.continent, dea.location, dea.date, dea.population, vac.new_vaccinations,
       SUM(CAST(vac.new_vaccinations AS INTEGER))
         OVER (PARTITION BY dea.location ORDER BY dea.location, dea.date) AS RollingPeopleVaccinated
FROM CovidDeaths dea
JOIN CovidVaccinations vac
  ON dea.location = vac.location
 AND dea.date = vac.date
WHERE dea.continent IS NOT NULL;

CREATE VIEW PercentPopulationVaccinated AS
SELECT dea.continent, dea.location, dea.date, dea.population, vac.new_vaccinations,
       SUM(CONVERT(INT, vac.new_vaccinations))
         OVER (PARTITION BY dea.location ORDER BY dea.location, dea.date) AS RollingPeopleVaccinated
FROM CovidDeaths dea
JOIN CovidVaccinations vac
  ON dea.location = vac.location
 AND dea.date = vac.date
WHERE dea.continent IS NOT NULL;
```

## Techniques Demonstrated
- Filtering and sorting (`WHERE`, `ORDER BY`)
- Aggregate functions (`SUM`, `MAX`, `COUNT`) with `GROUP BY`
- Percentage and ratio calculations with `CAST` for precision
- Table `JOIN`s across the deaths and vaccinations datasets
- Window functions (`SUM ... OVER (PARTITION BY ...)`) for rolling totals
- Common Table Expressions (CTEs)
- Temporary tables and views for reusable results
