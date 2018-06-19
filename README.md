坦克对战游戏 AI 设计

+ 参照师兄的代码来设计，模型采用官方商店下载的项目Tanks! Tutorial。该项目只保留模型

+ 使用NavMesh实现NPC坦克，有以下好处
    + 自动移动到目的地，可以不断设置新的目的地
    + 躲开不可走的地方
    + 自动实现模型的转向
    + 使用较为方便，只需在AI模型上添加Nav Mesh Agent组件
    + 方便区分障碍和可以走的路，需确保该模型有Mesh Renderer组件，在Navigation->Object->Navigation Area可以设置成可走/不可走/跳跃

+ 坦克，该类会派生出两个子类AI坦克和玩家坦克
    ```
    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;
    using UnityEngine.AI;

    public class Tank : MonoBehaviour {
    	private float hp; //生命值

    	public float getHp() {
    		return hp;
    	}

    	public void setHp(float hp) {
    		this.hp = hp; 
    	}

    }
    ```

+ AI坦克的设计：
    + 获取玩家位置信息后，具有以下行为：
        + 玩家离得远，AI坦克随机巡逻，单次巡逻超过5秒更换新目的地，防止卡住
        + 玩家离得近，AI坦克追逐玩家
        + 玩家进入AI坦克射程，AI坦克开火
        + 使用协程，每隔一秒开火一次
    
    + Enemy脚本：
    ```
    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;
    using UnityEngine.AI;

    public class Enemy : Tank {//npc坦克
	    public delegate void recycle(GameObject Tank);
	    public static event recycle recycleEvent;//npc坦克被销毁之后，可通知工厂回收
	
	    private Vector3 target;//目标，即玩家坦克的位置
        private Vector3 temp;//目前位置
        private int time;//该次巡逻时间
        public bool entry;

	    private bool gameover;//游戏是否结束，决定是否继续运动或射击

	    void Start() {
	    	setHp(100f);//设置初始生命值为100	
	    	StartCoroutine(shoot());//开始射击的协程
            newPoint();
            time = 0;
	    }

	    void Update () {
	    	gameover = GameDirector.getInstance().currentSceneController.isGameOver();
            if (!gameover) {
			    target = GameDirector.getInstance().currentSceneController.getPlayerPos();		
		    	if (getHp() <= 0 && recycleEvent != null) {//如果npc坦克被摧毁，则回收它
	    			recycleEvent(this.gameObject);
    			} else {//否则向玩家坦克移动或者随机移动
                    if (Vector3.Distance(transform.position, target) < 30)
                    {
                        NavMeshAgent agent = GetComponent<NavMeshAgent>();
                        agent.SetDestination(target);
                    }
                    else
                    {
                        time++;
                        if(time == 300)//此次巡逻时间达到5秒，更换目的地
                        {
                            newPoint();
                            time = 0;
                        }
                        if (transform.position.x == temp.x && transform.position.z == temp.z)//到达目的地，更换目的地
                        {
                            newPoint();
                            time = 0;
                        }
                        NavMeshAgent agent = GetComponent<NavMeshAgent>();
                        agent.SetDestination(temp);
                    }
		    	}
		    } else {//游戏结束，停止寻路
		    	NavMeshAgent agent = GetComponent<NavMeshAgent>();
		    	agent.velocity = Vector3.zero;
		    	agent.ResetPath();
		    }
		
	    }

        private void newPoint()
        {
            float x = Random.Range(-49, 49);
            float z = Random.Range(-49, 49);
            temp = new Vector3(x, transform.position.y, z);//随机生成目的地
        }

	    IEnumerator shoot() {//协程实现npc坦克每隔1s进行射击
		    while(!gameover) {
			    for (float i = 1; i > 0; i -= Time.deltaTime) {
			    	yield return 0;	
			    }
			    if (Vector3.Distance(transform.position, target) < 20) {//和玩家坦克距离小于20，则射击
                    myFactory mF = Singleton<myFactory>.Instance;
                    GameObject bullet = mF.getBullet(tankType.Enemy);//获取子弹，传入的参数表示发射子弹的坦克类型
                    bullet.transform.position = new Vector3(transform.position.x, 1.5f, transform.position.z) + transform.forward * 1.5f;//设置子弹
                    bullet.transform.forward = transform.forward;//设置子弹方向
                    Rigidbody rb = bullet.GetComponent<Rigidbody>();
                    rb.AddForce(bullet.transform.forward * 20, ForceMode.Impulse);//发射子弹
                }
		    }
	    }
    }
    ```

+ 玩家坦克
    ```
    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;
    using UnityEngine.AI;

    public class Player : Tank, IUserAction {//玩家坦克
	    public delegate void destroy();
	    public static event destroy destroyEvent;
        public GameObject player;

	    void Start() {
	    	setHp(500f);//设置初始生命值为500
            player = GameDirector.getInstance().currentSceneController.mF.getPlayer();
        }
	    void Update () {
            // 相机跟随玩家坦克
            Camera.main.transform.position = new Vector3(player.transform.position.x, 25, player.transform.position.z);
            if (getHp() <= 0 ) {//生命值<=0,表示玩家坦克被摧毁
	    		this.gameObject.SetActive(false);
	    		if (destroyEvent != null) {//执行委托事件
	    			destroyEvent();
	    		}
	    	}
	    }

        public Vector3 getPlayerPos()
        {//返回玩家坦克的位置
            return player.transform.position;
        }

        public void moveForward()
        {
            player.GetComponent<Rigidbody>().velocity = player.transform.forward * 20;
        }
        public void moveBackWard()
        {
            player.GetComponent<Rigidbody>().velocity = player.transform.forward * -20;
        }
        public void turn(float offsetX)
        {//通过水平轴上的增量，改变玩家坦克的欧拉角，从而实现坦克转向
            float x = player.transform.localEulerAngles.y + offsetX * 5;
            float y = player.transform.localEulerAngles.x;
            player.transform.localEulerAngles = new Vector3(y, x, 0);
            Camera.main.transform.localEulerAngles = new Vector3(y + 90, x, 0);
        }

        public void shoot()
        {
            GameObject bullet = GameDirector.getInstance().currentSceneController.mF.getBullet(tankType.Player);//获取子弹，传入的参数表示发出子弹的坦克类型
            bullet.transform.position = new Vector3(player.transform.position.x, 1.5f, player.transform.position.z) + player.transform.forward * 1.5f;//设置子弹位置
            bullet.transform.forward = player.transform.forward;//设置子弹方向
            Rigidbody rb = bullet.GetComponent<Rigidbody>();
            rb.AddForce(bullet.transform.forward * 20, ForceMode.Impulse);//发射子弹
        }
    }
    ```

+ 子弹
    ```
    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;

    public class Bullet : MonoBehaviour {
	    public float explosionRadius = 3f;//子弹的伤害半径
	    private tankType type;//发射子弹的坦克类型
	    void OnCollisionEnter(Collision other) {
		    myFactory mF = Singleton<myFactory>.Instance;
		    ParticleSystem explosion = mF.getPs();//获取爆炸的粒子系统
		    explosion.transform.position = transform.position;//设置粒子系统位置
		    Collider[] colliders = Physics.OverlapSphere(transform.position, explosionRadius);//获取爆炸范围内的所有碰撞体
		    for (int i = 0; i < colliders.Length; i++) {
		    	if (colliders[i].tag == "tankPlayer" && this.type == tankType.Enemy || colliders[i].tag == "tankEnemy" && this.type == tankType.Player) {//发射子弹的坦克类型和被击中的坦克类型不同，才造成伤害
		    		float distance = Vector3.Distance(colliders[i].transform.position, transform.position);//被击中坦克与爆炸中心的距离
		    		float hurt = 100f / distance;//受到的伤害
		    		float current = colliders[i].GetComponent<Tank>().getHp();//当前的生命值
		    		colliders[i].GetComponent<Tank>().setHp(current - hurt);//设置受伤后生命值
	    		}
	    	}
	    	explosion.Play();//播放爆炸的粒子系统
	    	if (this.gameObject.activeSelf) {
	    		mF.recycleBullet(this.gameObject);//回收子弹
	    	}
		
	    }

	    public void setTankType(tankType type) {//设置发射子弹的坦克类型
	    	this.type = type;
	    }
    }
    ```

+ 工厂，管理坦克跟子弹
    ```
    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;

    public enum tankType:int{Player, Enemy}
    public class myFactory : MonoBehaviour {//工厂
	    public GameObject player;//玩家坦克
	    public GameObject tank;//npc坦克
	    public GameObject bullet;//子弹
	    public ParticleSystem pS;//爆炸粒子系统
        public int count;

	    private Dictionary<int, GameObject> usingTanks;
	    private Dictionary<int, GameObject> freeTanks;
	    private Dictionary<int, GameObject> usingBullets;
	    private Dictionary<int, GameObject> freeBullets;

	    private List<ParticleSystem> psContainer;

	    void Awake() {
	    	usingTanks = new Dictionary<int, GameObject>();
	    	freeTanks = new Dictionary<int, GameObject>();
	    	usingBullets = new Dictionary<int, GameObject>();
	    	freeBullets = new Dictionary<int, GameObject>();
	    	psContainer = new List<ParticleSystem>();
	    }

	    void Start() {
	    	Enemy.recycleEvent += recycleTank;//npc坦克被摧毁时，会执行这个委托函数
            count = 0;
	    }

        public GameObject getPlayer() {//获取玩家坦克
	    	return player;
	    }

	    public GameObject getTank() {//获取npc坦克
	    	if (freeTanks.Count == 0) {
	    		GameObject newTank = Instantiate<GameObject>(tank);
	    		usingTanks.Add(newTank.GetInstanceID(), newTank);
	    		//在一个随机范围内设置坦克位置
	    		newTank.transform.position = new Vector3(Random.Range(-49, 49), 0, Random.Range(-49, 49));
	    		return newTank;
	    	}
	    	foreach (KeyValuePair<int, GameObject> pair in freeTanks) {
	    		pair.Value.SetActive(true);
	    		freeTanks.Remove(pair.Key);
	    		usingTanks.Add(pair.Key, pair.Value);
	    		pair.Value.transform.position = new Vector3(Random.Range(-49, 49), 0, Random.Range(-49, 49));
	    		return pair.Value;
	    	}
	    	return null;
	    }	

	    public GameObject getBullet(tankType type) {
	    	if (freeBullets.Count == 0) {
	    		GameObject newBullet = Instantiate(bullet);
	    		newBullet.GetComponent<Bullet>().setTankType(type);
	    		usingBullets.Add(newBullet.GetInstanceID(), newBullet);
	    		return newBullet;
	    	}
	    	foreach (KeyValuePair<int, GameObject> pair in freeBullets) {
	    		pair.Value.SetActive(true);
	    		pair.Value.GetComponent<Bullet>().setTankType(type);
	    		freeBullets.Remove(pair.Key);
	    		usingBullets.Add(pair.Key, pair.Value);
	    		return pair.Value;
	    	}
	    	return null;
	    }

	    public ParticleSystem getPs() {
	    	for (int i = 0; i < psContainer.Count; i++) {
	    		if (!psContainer[i].isPlaying) {
	    			return psContainer[i];
	    		}
	    	}
	    	ParticleSystem newPs = Instantiate<ParticleSystem>(pS);
	    	psContainer.Add(newPs);
	    	return newPs;
	    }

	    public void recycleTank(GameObject tank) {
	    	usingTanks.Remove(tank.GetInstanceID());
	    	freeTanks.Add(tank.GetInstanceID(), tank);
	    	tank.GetComponent<Rigidbody>().velocity = new Vector3(0, 0, 0);
	    	tank.SetActive(false);
            count++;
        }

	    public void recycleBullet(GameObject bullet) {
	    	usingBullets.Remove(bullet.GetInstanceID());
	    	freeBullets.Add(bullet.GetInstanceID(), bullet);
	    	bullet.GetComponent<Rigidbody>().velocity = new Vector3(0, 0, 0);
	    	bullet.SetActive(false);
	    }
	
        public bool noEnemy()
        {
            return usingTanks.Count == 0;
        }
    }
    ```

+ 获取用户输入
    ```
    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;

    public class UserAction : MonoBehaviour {
	    IUserAction action;
	    // Use this for initialization
	    void Start () {
	    	action = GameDirector.getInstance().currentSceneController.player as IUserAction;	
	    }
	
	    // Update is called once per frame
	    void Update () {
		    if (!GameDirector.getInstance().currentSceneController.isGameOver()) {
			    if (Input.GetKey(KeyCode.W)) {//向前
			    	action.moveForward();
			    }
			
			    if (Input.GetKey(KeyCode.S)) {//向后
			    	action.moveBackWard();
			    }

			    if (Input.GetKeyDown(KeyCode.Space)) {//射击
			    	action.shoot();	
			    }

    			float offsetX = Input.GetAxis ("Horizontal");//获取水平轴上的增量，目的在于控制玩家坦克的转向
    			action.turn(offsetX);
    		}
    	}
    }
    ```

+ 场记
    ```
    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;
    using UnityEngine.AI;

    public class SceneController : MonoBehaviour
    {
        public Player player;//玩家坦克

        private bool gameOver = false;//游戏是否结束 

        private int enemyConut = 6;//游戏的npc数量
        private myFactory mF;//工厂


        void Awake()
        {//一些初始的设置
            GameDirector director = GameDirector.getInstance();
            director.currentSceneController = this;
            mF = Singleton<myFactory>.Instance;
            player = Singleton<Player>.Instance;
        }
        void Start()
        {
            for (int i = 0; i < enemyConut; i++)
            {//获取npc坦克
                mF.getTank();
            }
            Player.destroyEvent += setGameOver;//如果玩家坦克被摧毁，则设置游戏结束
        }

        void Update()
        {
            if (mF.count == enemyConut)//如果敌人全部被摧毁，则设置游戏结束
            {
                setGameOver();
            }
        }

        public bool isGameOver()
        {//返回游戏是否结束
            return gameOver;
        }
        public void setGameOver()
        {//设置游戏结束
            gameOver = true;
        }

    }
    ```

+ 更多代码请下载项目查看（GameDirector、IUserAction、Singleton）

+ ![演示视频]()
