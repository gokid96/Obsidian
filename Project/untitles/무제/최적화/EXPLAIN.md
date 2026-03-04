-- 1. 폴더 트리 조회 (방금 N+1에서 쓴 쿼리)
EXPLAIN ANALYZE SELECT * FROM folder WHERE workspace_id = 6;

-- 2. 게시글 목록 조회
EXPLAIN ANALYZE SELECT * FROM post WHERE workspace_id = 6;

-- 3. 루트 게시글 조회
EXPLAIN ANALYZE SELECT * FROM post WHERE workspace_id = 6 AND folder_id IS NULL;

-- 4. 워크스페이스 멤버 수 카운트
EXPLAIN ANALYZE SELECT COUNT(*) FROM workspace_member WHERE workspace_id = 6;

-- 5. 폴더 수 카운트
EXPLAIN ANALYZE SELECT COUNT(*) FROM folder WHERE workspace_id = 6;