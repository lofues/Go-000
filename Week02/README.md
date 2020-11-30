1. 问题

        当DAO层出现sqlNoRows时错误时，err应当如何处理?
        
2. 思考解答
    
        当DAO层出现sqlNoRows时，在DAO层统一一个ErrNoFound的错误统一不同仓储实现时相同错误的场景;
        由于sqlNoRows是数据库响应没有数据时的特定err，是可预料之内的，因此不用wrap记录堆栈信息；需要wrap那些未知的err以便上层记录日志并追踪问题
        领域层对错误进行识别并处理，日志打印并构造用户可视化的错误信息
        应用层调用领域服务，具有报错时直接响应给用户，简化应用层逻辑便于聚合领域数据的聚合
        
3. 仓储层代码实现 

        var (
        	ErrNotFound = errors.New("record not found")
        )
        
        // 仓储层对象
        type RowUser struct {
        	Username    string
        	Id          string
        	PhoneNumber string
        }
        
        type Repo interface {
        	GetUserInfo(ctx context.Context, userId string) (*RowUser, error)
        }
        
        type repo struct {
        	SqlClient *sql.DB
        }
        
        func (r *repo) GetUserInfo(ctx context.Context, userId string) (*RowUser, error) {
        	u := RowUser{}
        	row := r.SqlClient.QueryRowContext(ctx, "SELECT phoneNumber, username FROM user WHERE id = ?;", userId)
        	err := row.Scan(&u.PhoneNumber, &u.Username)
        	if err != nil {
        	    // 对错误进行封装并打印记录堆栈等信息
        		if err == sql.ErrNoRows {
        			return nil, ErrNotFound
        		}
        		return nil, errors.Wrap(err, "查询用户失败")
        	}
        	return &u, nil
        }  

4. 领域层代码实现
      
       var (
            ErrUserNotFound = errors.New("用户不存在")
            ErrSystem = errors.New("系统出现异常,请稍后再试")
        )
      
       // 领域对象
       type User struct {
       	PhoneNumber string `json:"phoneNumber"`
       	UserName    string `json:"userName"`
       	Id          string `json:"id"`
       	Address     string `json:"address,omitempty"`
       }
       
       type Service interface {
       	GetUserInfo(ctx context.Context, userId string) (*User, error)
       }
       
       type service struct {
       	R      Repo
       }
       
       func NewService(r Repo) Service {
       	return &service{
       		R:      r,
       	}
       }
       
       func (s *service) GetUserInfo(ctx context.Context, userId string) (*User, error) {
       	ru, err := s.R.GetUserInfo(ctx, userId)
       	if err != nil {
       	    // 记录错误，构造可视化错误
       	    if errors.Is(err, ErrNotFound){
       	        return nil, ErrUserNotFound
       	    }
       	    // 记录未知错误日志，便于追踪问题
                    _ = logger.Log("err", err.Error())
       	     return nil, ErrSystem
       	}
       	// 转换成领域层对象返回
       	u := ru.ToUser()
       	return &u, nil
       }
 
5. 应用层代码实现

        // 应用层对象
        type UserInfoResponse struct{
            User
            Err string `json:"err,omitempty"`
        }
        
        type Application interface{
            GetUserInfo(ctx context.Context, userId string) (UserInfoResponse, error)
        }
        
        type application struct{
            // 领域服务
            S Service
        }
        
        func (a *Application) GetUserInfo(ctx context.Context, userId string) UserInfoResponse{
            res := UserInfoResponse{}
            u, err := a.S.GetUserInfo(ctx, userId)
            // 对错误直接响应给用户
            if err != nil{
                res.Err = err.Error()
                return res
            }
            u.User = u
            return u
        }
        
        