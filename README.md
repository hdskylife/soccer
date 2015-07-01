
from BeautifulSoup import BeautifulSoup 
2 import urllib2 
3 import re 
4 import csv 
5 import datetime 
6 import MySQLdb 
7 
 
8 
 
9 def getHtml(url): 
10     response = urllib2.urlopen(url) 
11     doc = response.read() 
12 
 
13     return doc 
14 
 
15 
 
16 def getResults(html): 
17     soup = BeautifulSoup(html) 
18 
 
19     results = soup.findAll('tr',attrs={'class':re.compile('[odd|even]row')}) 
20 
 
21     game_results = [] 
22     for r in results: 
23 
 
24         cells = r.findAll('td') 
25 
 
26         date_played         = getMatchDate(r) 
27         home_team           = cells[1].a.renderContents() 
28         away_team           = cells[3].a.renderContents() 
29         score               = cells[2].a.renderContents() 
30         home_score          = score.split('-')[0] 
31         away_score          = score.split('-')[1] 
32         stadium_attendance  = splitStadiumAndAttendance(cells[5].renderContents()) 
33         stadium             = stadium_attendance['stadium'] 
34         attendance          = stadium_attendance['attendance']  
35 
 
36         game_results.append((date_played, home_team, away_team, home_score, away_score, stadium, attendance)) 
37 
 
38     return game_results 
39 
 
40 
 
41 def getMatchDate(row): 
42      
43     raw_date = row.parent.tr.td.renderContents() 
44     parsed_date = datetime.datetime.strptime(raw_date, '%A, %B %d, %Y') 
45 
 
46     return parsed_date.date() 
47 
 
48 
 
49 def splitStadiumAndAttendance(str_stadium_attendance): 
50     matcha = re.search('<a.*>(?P<stadium>.+)</a>\s+\((?P<attendance>[0-9,]+)\)',str_stadium_attendance) 
51     if matcha: 
52         return matcha.groupdict() 
53     else: 
54         match = re.search('(?P<stadium>.+)\s+\((?P<attendance>[0-9,]+)\)',str_stadium_attendance) 
55         if match: 
56             return match.groupdict() 
57 
 
58 
 
59     return {'stadium':'unknown','attendance':'unknown'} 
60 
 
61 
 
62 def writeToCsv(filename, data): 
63     writer = csv.writer(open(filename,'ab'),lineterminator='\n',delimiter='\t') 
64     writer.writerows(data) 
65 
 
66 
 
67 def writeToDatabase(data, league_id): 
68     conn = MySQLdb.connect(user='nba',passwd='lakers',db='soccer') 
69     curs = conn.cursor() 
70 
 
71     for row in data: 
72         sql = """ 
73             INSERT IGNORE INTO matchresults  
74             (league_id, date_played, home_team, away_team, home_score, away_score, stadium, attendance) 
75             VALUES (%s) 
76         """ % ','.join([str(league_id)] + ['"%s"' % str(datapoint) for datapoint in row]) 
77         curs.execute(sql) 
78 
 
79 
 
80 def main(): 
81     dates = [ 
82         20110424,20110401,20110320,20110219,20110119, 
83         20101219,20101120,20101017,20101018,20100918,20100830,20100828 
84     ] 
85     dates = [ 
86         20110520,20110505,20110422,20110401,20110320,20110219,20110119, 
87         20101219,20101120,20101017,20101018,20100918,20100830,20100828 
88     ] 
89     #base_url = 'http://soccernet.espn.go.com/results/_/league/ita.1/date/<date>/italian-serie-a?cc=5901' 
90     #base_url = 'http://soccernet.espn.go.com/results/_/league/eng.1/date/<date>/barclays-premier-league?cc=5901' 
91     base_url = 'http://soccernet.espn.go.com/results/_/league/esp.1/date/<date>/spanish-la-liga?cc=5901' 
92     #base_url = 'http://soccernet.espn.go.com/results/_/league/ger.1/date/<date>/german-bundesliga?cc=5901' 
93 
 
94     all_results = [] 
95     for dt in dates: 
96         url = base_url.replace('<date>',str(dt)) 
97 
 
98         print url 
99         html = getHtml(url) 
100         all_results.extend(getResults(html)) 
101         #writeToCsv('serie_a_results.txt',results) 
102 
 
103     all_results = sorted(list(set(all_results))) 
104     writeToDatabase(all_results, 2) 
105 
 
106 
 
107 
 
108 if __name__ == '__main__': 
109     main() 
