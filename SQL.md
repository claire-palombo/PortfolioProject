/*  SQL Project

    Claire Palombo */

# Create team table

CREATE TABLE `G12`.`team` (

`teamID`	TINYINT NOT NULL AUTO_INCREMENT,

`teamName`	VARCHAR(25)	NOT NULL,

PRIMARY KEY (`teamID`));

# Create league table

CREATE TABLE `G12`.`league` (

`leagueID`	TINYINT NOT NULL AUTO_INCREMENT,

`leagueName`	VARCHAR(25)	NOT NULL,

PRIMARY KEY	(`leagueID`));

# Create referee table

CREATE TABLE `G12`.`referee` (

`refID`	TINYINT NOT NULL	AUTO_INCREMENT,

`refName`	VARCHAR(25)	NULL,

PRIMARY KEY (`refID`));

# Insert values team table

INSERT INTO G12.team(teamName)

SELECT DISTINCT homeTeam AS Team 

FROM(

	SELECT homeTeam

    FROM fcp_2023.results_csv AS h

    UNION

    SELECT awayTeam

    FROM fcp_2023.results_csv AS a

) as t

ORDER BY Team;

# Insert values league table

INSERT INTO G12.league(leagueName)

SELECT DISTINCT league

FROM fcp_2023.results_csv

ORDER BY league;

# Insert values referee table

INSERT INTO G12.referee(refName)

SELECT referee

FROM fcp_2023.results_csv

GROUP BY referee

ORDER BY referee;

# Create season table

CREATE TABLE G12.season (

`seasonID`	TINYINT NOT NULL AUTO_INCREMENT,

`seasonName`	CHAR(9)	NOT NULL ,

PRIMARY KEY (`seasonID`),

UNIQUE(`seasonName`));

#Create Temp Table for Home and Away Teams

CREATE TABLE `G12`.`temp`(

 `fixtureID` TINYINT NOT NULL AUTO_INCREMENT,

 `homeTeam` VARCHAR(25) NOT NULL,

 `awayTeam` VARCHAR(25) NOT NULL,

 `date` DATE NOT NULL,

 `time` TIME NOT NULL,

 PRIMARY KEY(fixtureID))

#Insert Data into temp table:

INSERT INTO G12.temp(homeTeam,awayTeam,`date`,`time`) 

SELECT homeTeam, awayTeam, `date`, `time`

FROM fcp_2023.results_csv

#Create fixture table

CREATE TABLE G12.fixture (

`fixtureID`	TINYINT NOT NULL REFERENCES `G12`.`temp`(fixtureID),

`seasonID`	TINYINT NOT NULL ,

`refID`	TINYINT NOT NULL ,

`date`	DATE NOT NULL,

`time`	TIME NOT NULL,

`htr`     CHAR(1) NOT NULL,

`ftr`     CHAR(1) NOT NULL, 

PRIMARY KEY (`fixtureID`),

CONSTRAINT `seasonID_fk` FOREIGN KEY (`seasonID`)

      REFERENCES `G12`.`season` (`seasonID`)

      ON DELETE NO ACTION ON UPDATE NO ACTION,

CONSTRAINT `refID_fk` FOREIGN KEY (`refID`)

      REFERENCES `G12`.`referee` (`refID`)

      ON DELETE NO ACTION ON UPDATE NO ACTION);

#Insert data season

INSERT INTO G12.season(seasonName)

SELECT DISTINCT season

FROM fcp_2023.results_csv

ORDER BY 1;

#Insert data fixture

INSERT INTO G12.fixture(fixtureID, seasonID, refID, date, time, htr, ftr)

SELECT 

d.fixtureID,

b.seasonID, 

c.refID,

a.date,

a.time,

a.htr,

a.ftr

from fcp_2023.results_csv a, G12.season b, G12.referee c, G12.temp d

WHERE a.season = b.seasonName AND a.referee = c.refName AND concat(a.homeTeam,a.awayTeam,a.`date`,a.`time`) = concat(d.homeTeam,d.awayTeam,d.`date`,d.`time`)

#Create division table

CREATE TABLE G12.division 

(`divisionID` TINYINT NOT NULL AUTO_INCREMENT, `divisionName` VARCHAR(50) NOT NULL,

`leagueID` TINYINT  NOT NULL, 

PRIMARY KEY(divisionID),

FOREIGN KEY (leagueID) REFERENCES `G12`.`league`(`leagueID`)

ON DELETE NO ACTION ON UPDATE NO ACTION);

#Create team_season_div

CREATE TABLE G12.team_season_div 

(`teamID` TINYINT NOT NULL, 

`seasonID` TINYINT NOT NULL, 

divisionID TINYINT NOT NULL,

PRIMARY KEY (teamID, seasonID),

CONSTRAINT team_ID_fk 

FOREIGN KEY (teamID) 

REFERENCES `team`(`teamID`) ON DELETE NO ACTION ON UPDATE NO ACTION,

CONSTRAINT season_ID_fk 

FOREIGN KEY (seasonID) 

REFERENCES `season`(`seasonID`) ON DELETE NO ACTION ON UPDATE NO ACTION,

CONSTRAINT division_ID_fk 

FOREIGN KEY (divisionID) 

REFERENCES `division`(`divisionID`) ON DELETE NO ACTION ON UPDATE NO ACTION);

#Insert data into division table

INSERT INTO G12.division(divisionName, leagueID)

SELECT DISTINCT `div` AS divisionName, l.leagueID

FROM fcp_2023.results_csv a

JOIN G12.league l ON a.league = l.leagueName

#Insert data into team_season_div 

INSERT IGNORE INTO G12.team_season_div (teamID, seasonID, divisionID)

SELECT t.teamID, s.seasonID, d.divisionID

FROM fcp_2023.results_csv a

JOIN G12.team t ON a.homeTeam = t.teamName

JOIN G12.team ta ON a.awayTeam = ta.teamName

JOIN G12.season s ON a.season = s.seasonName

JOIN G12.division d ON a.`div` = d.divisionName

ORDER BY t.teamID

#Create team_fixture table

CREATE TABLE G12.team_fixture 

(`teamID` TINYINT NOT NULL REFERENCES  `team`(`teamID`) ON DELETE NO ACTION ON UPDATE NO ACTION , 

`fixtureID` TINYINT NOT NULL REFERENCES `fixture`(`fixtureID`) ON DELETE NO ACTION ON UPDATE NO ACTION,

`isHome` CHAR(1) NOT NULL, 

`shots` TINYINT NOT NULL,

`shotsTarget` TINYINT NOT NULL,

`corners` TINYINT NOT NULL,

`fouls` TINYINT NOT NULL,

`yellowCards` TINYINT NOT NULL,

`redCards` TINYINT NOT NULL, 

`goalsHT` TINYINT NOT NULL,

`goalsFT` TINYINT NOT NULL,

PRIMARY KEY(teamID,FixtureID));

#Insert Data into team_fixture table

INSERT INTO G12.team_fixture(teamID,fixtureID,isHome,shots,shotsTarget,corners,

fouls,yellowCards, redCards,goalsHT,goalsFT)

SELECT teamID, fixtureID, 'Y' AS isHome, r.hs AS shots, r.hst AS shotsTarget,

r.hc AS corners, r.hf AS fouls, r.hy AS yellowCards, r.hr AS redCards,

r.hthg AS goalsHT, r.fthg AS goalsFT

FROM fcp_2023.results_csv r

JOIN team a ON r.awayTeam = a.teamName

JOIN temp b ON a.teamName = b.awayTeam

JOIN fcp_2023.results_csv  ON concat(r.awayTeam,r.date,r.time) = concat(b.awayTeam,b.date,b.time)

Union

SELECT teamID, fixtureID, 'N' AS isHome, r.as AS shots, r.ast AS shotsTarget,

r.ac AS corners, r.af AS fouls, r.ay AS yellowCards, r.ar AS redCards,

r.htag AS goalsHT, r.ftag AS goalsFT

FROM fcp_2023.results_csv r

JOIN team a ON r.homeTeam = a.teamName

JOIN temp b ON a.teamName = b.homeTeam

JOIN fcp_2023.results_csv  ON concat(r.homeTeam,r.date,r.time) = concat(b.homeTeam,b.date,b.time)

ORDER BY fixtureID;

SELECT fixtureID, 

       hometeamID AS TeamID,

       'Y' AS IsHomeTeam

FROM (

    SELECT fixtureID, b.TeamID as hometeamID, 'Y' AS IsHomeTeam

    FROM temp a

    JOIN (

        SELECT DISTINCT TeamID, teamName 

        FROM team a 

        JOIN temp b ON a.teamName = b.homeTeam

    ) b ON a.hometeam = b.teamName

) AS HomeTeams

UNION

SELECT fixtureID, 

       awayteamID AS TeamID,

       'N' AS IsHomeTeam

FROM (

    SELECT fixtureID, c.TeamID as awayteamID, 'N' AS IsHomeTeam

    FROM temp a

    JOIN (

        SELECT DISTINCT TeamID, teamName 

        FROM team a

        JOIN temp b ON a.teamName = b.awayTeam

    ) c ON a.awayteam = c.teamName

) AS AwayTeams

ORDER BY fixtureID

SELECT DISTINCT TeamID, fixtureID, c.`as` as shots, c.ast as shotsTarget, c.ac as corners,

 c.af as fouls, c.ay as yellowCards, c.ar as redCards, c.htag as goalsHT, c.ftag as goalsFT

        FROM team a

        JOIN temp b ON a.teamName = b.awayTeam

        JOIN fcp_2023.results_csv c ON concat(c.awayTeam,c.date,c.time) = concat(b.awayTeam,b.date,b.time)

        

UNION

SELECT DISTINCT TeamID, fixtureID, c.hs as shots, c.hst as shotsTarget, c.hc as corners, 

c.hf as fouls, c.hy as yellowCards, c.hr as redCards, c.hthg as goalsHT, c.fthg as goalsFT

        FROM team a

        JOIN temp b ON a.teamName = b.homeTeam

        JOIN fcp_2023.results_csv c ON concat(c.homeTeam,c.date,c.time) = concat(b.homeTeam,b.date,b.time)        

        

ORDER BY date, time
