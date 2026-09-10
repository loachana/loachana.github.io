%------- CAMPUS SAFETY ROUTE FINDER SYSTEM ----------


:- dynamic(blocked/2).



%------------ campus map (edges with distances) ------------
edge(main_entrance, parking_area, 5).
edge(main_entrance, auditorium, 9).
edge(main_entrance, it_lab2, 10).
edge(main_entrance, admin_building, 13).
edge(parking_area, auditorium, 3).
edge(auditorium, canteen, 15).
edge(auditorium, student_union_office, 9).
edge(it_lab2, student_union_office, 4).
edge(admin_building, lecture_hall2, 13).
edge(canteen, library, 12).
edge(canteen, study_area, 11).
edge(canteen, student_union_office, 4).
edge(student_union_office, it_lab1, 17).
edge(it_lab1, lecture_hall1, 10).
edge(it_lab1, lecture_hall2, 10).
edge(lecture_hall2, lecture_hall1, 18).
edge(lecture_hall1, study_area, 18).
edge(lecture_hall1, medical_center, 10).
edge(medical_center, study_area, 7).
edge(medical_center, hostel, 10).
edge(hostel, study_area, 8).
edge(hostel, library, 25).
edge(library, gym, 12).


%---------- connections (undirected) ------------
connected(A,B,D):- edge(A,B,D), \+ blocked(A,B).
connected(A,B,D):- edge(B,A,D), \+ blocked(B,A).


%----------- Heuristic Values (straight-line estimates to goal gym) ---------------

h(main_entrance, 35).
h(parking_area, 32).
h(auditorium, 28).
h(it_lab2, 30).
h(admin_building, 25).
h(canteen, 20).
h(student_union_office, 22).
h(it_lab1, 26).
h(lecture_hall1, 18).
h(lecture_hall2, 20).
h(library, 12).
h(study_area, 15).
h(medical_center, 14).
h(hostel, 20).
h(gym, 0).       % goal


%------------ Main Menu -----------
go:-
	nl,write('========= Campus Safety Route Finder System =========='),nl,
	write('Welcome! This system finds safe paths between campus locations.'), nl, nl,
	write('1. Find path'), nl,
	write('2. Block a road'), nl,
	write('3. Unblock a road'), nl,
	write('4. Show blocked roads'), nl,
	write('5. Exit'), nl, nl,
	write('Choosen an option (1-5): '),read(Choice),
	handle(Choice).

handle(1):- 
	nl, write('Enter Start Location: '), read(Start),
	write('Enter Goal Location: '), read(Goal),
	show_all_paths(Start, Goal),
	 go.

handle(2):-
	nl, write('Enter Road to Block(example: canteen. student_union_office.): '),
	read(A), read(B),
	assert(blocked(A,B)), nl,
	write('Road blocked: '), write(A-B), nl, 
	go.


handle(3):-
	nl, write('Enter Road to Unblock (example: canteen. student_union_office.): '),
	read(A), read(B),
	(retract(blocked(A,B)) -> write('Unblocked: '), nl,
	write(A-B), nl ; nl, write('No such blocked road found.'), nl),
	go.


handle(4):-
	nl, write('Currently blocked roads: '), nl, nl,
	(blocked(A,B) -> list_blocked ; write('None'), nl),
	go.

handle(5):-
	write('Goodbye!'), nl.

handle(_):- 
	write('Invalid choice, try again.'), nl,
	go.


list_blocked:- forall(blocked(A,B),(write(A-B), nl)).


%----------DFS (Find All Paths with Cost) ---------
dfs_path(Start, Goal, Path, Cost):-
	dfs_travel(Start, Goal, [Start], RevPath, 0, Cost),
	reverse(RevPath, Path).

dfs_travel(Node, Node, Path, Path, Cost, Cost).
dfs_travel(Current, Goal, Visited, Path, CostSoFar, Cost):-
	connected(Current, Next, StepCost),
	\+ member(Next, Visited),
	NewCost is CostSoFar + StepCost,
	dfs_travel(Next, Goal, [Next|Visited], Path, NewCost, Cost).



%---------- BFS (Shortest Path in terms of steps) ---------------
bfs(Start, Goal, Path, Cost):-
	bfs_queue([[Start]], Goal, RevPath),
	reverse(RevPath, Path),
	path_cost(Path,Cost).

bfs_queue([[Goal|Rest]|_], Goal, [Goal|Rest]).
bfs_queue([[Current|Rest]|Other], Goal, Path) :-
	findall([Next,Current|Rest],
	(connected(Current, Next, _),
	\+ member(Next, [Current|Rest])),
	NewPaths),
	append(Other, NewPaths, Updated),
	bfs_queue(Updated, Goal, Path).



%--------------- A* Search (Shortest Path in terms of cost) -------------
astar(Start, Goal, Path, Cost):-
	h(Start, H0),
	astar_search([[H0,0,[Start]]], Goal, RevPath, Cost),
	reverse(RevPath, Path).

astar_search([[_,Cost,[Goal|Rest]]|_], Goal, [Goal|Rest], Cost).
astar_search([[F,G,[Current|Rest]]|Others], Goal, Path, Cost):-
	findall([F2, G2,[Next,Current|Rest]],
		(connected(Current, Next, StepCost),
		\+ member(Next,[Current|Rest]),
		G2 is G + StepCost,
		h(Next,H),
		F2 is G2 + H),
		Children),
	append(Others, Children, All),
	sort(All,Sorted),
	astar_search(Sorted,Goal,Path,Cost).



%-------------- Path Cost Helper -----------
path_cost([_],0).
path_cost([A,B|Rest],Cost):-
	connected(A,B,D),
	path_cost([B|Rest],CostRest),
	Cost is D + CostRest.




%------------- Display Results -------------
show_all_paths(Start, Goal):-
	findall([Pdfs,Cdfs],dfs_path(Start, Goal, Pdfs, Cdfs), DfsPaths),
	(bfs(Start, Goal, Pbfs, Cbfs) -> true ; (Pbfs=[],Cbfs=0)),
	(astar(Start, Goal, Pa, Ca) -> true ; (Pa=[],Ca=0)),

	(DfsPaths = [] -> write('No path found!'), nl
	;
	write('------------------------------'), nl,
	write('------------------------------'), nl,
	write('All paths :'), nl,
	display_paths(DfsPaths),
	write('------------------------------'), nl,
	write('Safe Path chosen: '), write(Pa), nl,
	write('Total Cost: '), write(Ca), nl
	).



display_paths([]).
display_paths([[P,C]|Rest]) :-
	write('Path= '), write(P), nl,
	write(' cost= '), write(C), nl, nl,
	display_paths(Rest).




