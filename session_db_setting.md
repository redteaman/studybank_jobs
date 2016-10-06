# PHP Session DB Settings

1. 確認 PHP ext 中是否有安裝 PDO 套件
2. 修改 PHP ini 將 session.save_handler 由 files 改為 user
3. 新增 SESSION 存取用 DB 資料表

```
CREATE TABLE [dbo].[sessions](
	[id] [varchar](32) NOT NULL,
	[access] [int] NOT NULL,
	[data] [ntext] NOT NULL,
 CONSTRAINT [PK_sessions] PRIMARY KEY CLUSTERED 
(
	[id] ASC
)WITH (PAD_INDEX  = OFF, STATISTICS_NORECOMPUTE  = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS  = ON, ALLOW_PAGE_LOCKS  = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
```

4. 準備 SESSION 存取程式

```
//設置垃圾回收最大生存時間
ini_set('session.gc_maxlifetime', 1440);
//設置用戶自定義Session存儲
ini_set('session.save_handler', 'user');
// PDO 函式庫
include_once("pdo/Db.class.php");

class session
{
    // session-lifetime
    var $lifeTime;
    public $pdodb;

    function open($savePath, $sessName)
    {
        $this->pdodb = new Db();
        // get session-lifetime
        $this->lifeTime = get_cfg_var("session.gc_maxlifetime");
        return true;
    }

    function close()
    {
        $this->gc(ini_get('session.gc_maxlifetime'));
        return true;
    }

    function read($sessID)
    {
        // fetch session-data
        $para = array('id' => $sessID, 'access' => time());
        $rs = $this->pdodb->query("SELECT data FROM sessions WHERE id = :id AND access > :access", $para);
        // return data or an empty string at failure
        if (!empty($rs)) {
            return $rs[0]['data'];
        } else {
            return false;
        }
    }

    function write($sessID, $sessData)
    {
        // new session-expire-time
        $newExp = time() + $this->lifeTime;
        // fetch session-data
        $para = array('id' => $sessID, 'access' => time());
        $data = $this->pdodb->query("SELECT * FROM sessions WHERE id = :id AND access > :access", $para);
        // return data or an empty string at failure
        if (isset($data[0]['id'])) {
            // ...update session-data
            $para = array('data' => $sessData, 'id' => $sessID);
            $this->pdodb->query("UPDATE sessions SET data = :data WHERE id = :id", $para);
            return true;
        } else {
            // create a new row
            $para = array('data' => $sessData, 'id' => $sessID, 'access' => $newExp);
            $this->pdodb->query("INSERT INTO sessions Values(:id, :access, :data)", $para);
            return true;
        }
        // an unknown error occured
        return false;
    }
    function destroy($sessID)
    {
        // delete session-data
        $result = $this->pdodb->query("DELETE sessions WHERE id = :id", array('id' => $sessID));
        if ($result) {  // if session was deleted, return true,
            return true;
        } else {    // ...else return false
            return false;
        }
    }

    function gc($sessMaxLifeTime)
    {
        // delete old sessions
        $result = $this->pdodb->query("DELETE sessions WHERE access < :access", array('access' => time()));
        if ($result) {  // return affected rows
            return true;
        } else {
            return false;
        }
    }

}

$session = new session();
//  5.3 版可用
session_set_save_handler(array(&$session, "open"),
    array(&$session, "close"),
    array(&$session, "read"),
    array(&$session, "write"),
    array(&$session, "destroy"),
    array(&$session, "gc"));
```

5. 修改 session_start() 使用檔案，將上述檔案 include 進去，才進行 session_start()
