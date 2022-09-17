# SwiftyStoreKitSubscriptions

### FETCH SERVICE

```swift
//
//  IAPFetchService.swift
//  DailyWeightMeasure
//
//  Created by paige shin on 2022/09/14.
//

import Foundation
import Combine
import StoreKit
import SwiftyStoreKit

enum IAPFetchProductServiceError: Error {
    case networkNotConnected // Handle UI
    case fetchProductsError(Error)
    case invalidProductsIds
    case unknown
}

extension IAPFetchProductServiceError: LocalizedError {
    var errorDescription: String? {
        switch self {
        case .networkNotConnected:
            return "Network is not connected"
        case .fetchProductsError(let error):
            return error.localizedDescription
        case .invalidProductsIds:
            return "Invalid Product Ids..."
        case .unknown:
            return "Unknown Error while fetching products..."
        }
    }
}

protocol IAPFetchProductServiceProtocol {
    init(networkConnectivity: NetworkConnectivityProtocol,
         productIds: Set<String>)
    func fetchProducts() -> AnyPublisher<Set<SKProduct>, IAPFetchProductServiceError>
}

struct IAPFetchProductService: IAPFetchProductServiceProtocol {
    
    private var networkConnectivity: NetworkConnectivityProtocol
    private var productIds: Set<String>
    
    init(networkConnectivity: NetworkConnectivityProtocol, productIds: Set<String>) {
        self.networkConnectivity = networkConnectivity
        self.productIds = productIds
    }
    
    func fetchProducts() -> AnyPublisher<Set<SKProduct>, IAPFetchProductServiceError> {
        Deferred {
            Future { promise in
                if !self.networkConnectivity.isConnectedToNetwork() {
                    printd(loggingLevel: .error(IAPFetchProductServiceError.networkNotConnected))
                    promise(.failure(.networkNotConnected))
                    return
                }
                SwiftyStoreKit.retrieveProductsInfo(self.productIds) { result in

                    if let error: Error = result.error {
                        printd(loggingLevel: .error(error))
                        printd(loggingLevel: .error(IAPFetchProductServiceError.fetchProductsError(error)))
                        promise(.failure(.fetchProductsError(error)))
                        return
                    }

                    if result.retrievedProducts.count > 0 {
                        let products: Set<SKProduct> = result.retrievedProducts
                        promise(.success(products))
                        return
                    }
                    
                    if result.invalidProductIDs.count > 0 {
                        printd(loggingLevel: .error(IAPFetchProductServiceError.invalidProductsIds))
                        promise(.failure(.invalidProductsIds))
                        return
                    }

                    printd(loggingLevel: .error(IAPFetchProductServiceError.unknown))
                    promise(.failure(.unknown))

                }
            }
        }
        .eraseToAnyPublisher()

    }
    
    
}

```

### Purchase Service

```swift
//
//  IAPPurchaseService.swift
//  DailyWeightMeasure
//
//  Created by paige shin on 2022/09/14.
//

import Foundation
import Combine
import StoreKit
import SwiftyStoreKit

enum IAPPurchaseServiceError: Error {
    // Purchase
    case paymentError(Error)
    case clientInvalid
    case paymentCancelled // Dont Handle
    case paymentInvalid
    case paymentNotAllowed
    case storeProductNotAvailable
    case cloudServicePermissionDenied
    case cloudServiceNetworkConnectionFailed
    case cloudServiceRevoked
    case uncategorized(Error)
    case deferred(PurchaseDetails)
    case refreshReceiptError(Error)
    case networkNotConnected // Handle UI
    case unknown
    case notPurchased
    case expired(productId: String, expiryDate: Date, receiptItems: [ReceiptItem]) // Handle UI
    case receiptVerificationFailed(String, Error)
}


extension IAPPurchaseServiceError: LocalizedError {
    var errorDescription: String? {
        switch self {
        case .networkNotConnected:
            return "Network is not connected"
        case .paymentError(let error):
            return error.localizedDescription
        case .unknown:
            return "Unknown error. Please contact support"
        case .clientInvalid:
            return "Not allowed to make the payment"
        case .paymentCancelled:
            return "Payment Cancelled"
        case .paymentInvalid:
            return "The purchase identifier was invalid"
        case .paymentNotAllowed:
            return "The device is not allowed to make the payment"
        case .storeProductNotAvailable:
            return "The product is not available in the current storefront"
        case .cloudServicePermissionDenied:
            return "Access to cloud service information is not allowed"
        case .cloudServiceNetworkConnectionFailed:
            return "Could not connect to the network"
        case .cloudServiceRevoked:
            return "User has revoked permission to use this cloud service"
        case .uncategorized(let error):
            return error.localizedDescription
        case .deferred(let purchase):
            return "deferred: \(purchase)"
        case .expired(let productId, let date, let receiptItems):
            return "\(productId) is expired since \(date)\n\(receiptItems)\n"
        case .notPurchased:
            return "Product Not Purchased"
        case .receiptVerificationFailed(let productId, let error):
            return "\(productId): \(error.localizedDescription)"
        case .refreshReceiptError(let error):
            return error.localizedDescription
        }
    }
}

typealias PurchaseResult = (productId: String, expiryDate: Date, receipts: [ReceiptItem])

protocol IAPPurchaseServiceProtocol {
    
    init(networkConnectivity: NetworkConnectivityProtocol, sharedSecret: String)
    func purchase(productId: String) -> AnyPublisher<PurchaseResult, IAPPurchaseServiceError>
}

struct IAPPurchaseService: IAPPurchaseServiceProtocol {
    
    private var networkConnectivity: NetworkConnectivityProtocol
    private var sharedSecret: String
    
    init(networkConnectivity: NetworkConnectivityProtocol, sharedSecret: String) {
        self.networkConnectivity = networkConnectivity
        self.sharedSecret = sharedSecret
    }
    
    func purchase(productId: String) -> AnyPublisher<PurchaseResult, IAPPurchaseServiceError> {
        Deferred {
            Future { promise in
                if !self.networkConnectivity.isConnectedToNetwork() {
                    printd(loggingLevel: .error(IAPPurchaseServiceError.networkNotConnected))
                    promise(.failure(.networkNotConnected))
                    return
                }
                SwiftyStoreKit.purchaseProduct(productId, quantity: 1, atomically: false) { result in
                    switch result {
                    case .success(let purchase):
                        printd("PURCHASE SUCCEEDED!")
                        self.verifySubscription(productId: purchase.productId, promise: promise)
                    case .error(let error):
                        printd(loggingLevel: .error(error))
                        switch error.code {
                        case .unknown:
                            printd(loggingLevel: .error(IAPPurchaseServiceError.unknown))
                            promise(.failure(.unknown))
                        case .clientInvalid:
                            printd(loggingLevel: .error(IAPPurchaseServiceError.clientInvalid))
                            promise(.failure(.clientInvalid))
                        case .paymentCancelled:
                            printd(loggingLevel: .error(IAPPurchaseServiceError.paymentCancelled))
                            promise(.failure(.paymentCancelled))
                        case .paymentInvalid:
                            printd(loggingLevel: .error(IAPPurchaseServiceError.paymentInvalid))
                            promise(.failure(.paymentInvalid))
                        case .paymentNotAllowed:
                            printd(loggingLevel: .error(IAPPurchaseServiceError.paymentNotAllowed))
                            promise(.failure(.paymentNotAllowed))
                        case .storeProductNotAvailable:
                            printd(loggingLevel: .error(IAPPurchaseServiceError.storeProductNotAvailable))
                            promise(.failure(.storeProductNotAvailable))
                        case .cloudServicePermissionDenied:
                            printd(loggingLevel: .error(IAPPurchaseServiceError.cloudServicePermissionDenied))
                            promise(.failure(.cloudServicePermissionDenied))
                        case .cloudServiceNetworkConnectionFailed:
                            printd(loggingLevel: .error(IAPPurchaseServiceError.cloudServiceNetworkConnectionFailed))
                            promise(.failure(.cloudServiceNetworkConnectionFailed))
                        case .cloudServiceRevoked:
                            printd(loggingLevel: .error(IAPPurchaseServiceError.cloudServiceRevoked))
                            promise(.failure(.cloudServiceRevoked))
                        default:
                            printd(loggingLevel: .error(IAPPurchaseServiceError.uncategorized(error)))
                            promise(.failure(.uncategorized(error)))
                        }
                    case .deferred(purchase: let purchase):
                        printd(loggingLevel: .error(IAPPurchaseServiceError.deferred(purchase)))
                        promise(.failure(.deferred(purchase)))
                    }
                }
            }
        }
        .eraseToAnyPublisher()
    }
    
    private func verifySubscription(productId: String, promise: @escaping(Result<PurchaseResult, IAPPurchaseServiceError>) -> Void) {
        let appleValidator: AppleReceiptValidator
        #if DEBUG
            appleValidator = AppleReceiptValidator(service: .sandbox, sharedSecret: self.sharedSecret)
        #else
            appleValidator = AppleReceiptValidator(service: .production, sharedSecret: self.sharedSecret)
        #endif
        SwiftyStoreKit.verifyReceipt(using: appleValidator) { result in
            switch result {
            case .success(let receipt):
                // Verify the purchase of a Subscription
                let purchaseResult = SwiftyStoreKit.verifySubscription(
                    ofType: .autoRenewable, // or .nonRenewing (see below)
                    productId: productId,
                    inReceipt: receipt)
                switch purchaseResult {
                case .purchased(let expiryDate, let items):
                    promise(.success((productId, expiryDate, items)))
                case .expired(let expiryDate, let items):
                    printd(loggingLevel: .error(IAPPurchaseServiceError.expired(productId: productId, expiryDate: expiryDate, receiptItems: items)))
                    promise(.failure(.expired(productId: productId, expiryDate: expiryDate, receiptItems: items)))
                case .notPurchased:
                    printd(loggingLevel: .error(IAPPurchaseServiceError.notPurchased))
                    promise(.failure(.notPurchased))
                }
            case .error(let error):
                printd(loggingLevel: .error(error))
                printd(loggingLevel: .error(IAPPurchaseServiceError.receiptVerificationFailed(productId, error)))
                promise(.failure(.receiptVerificationFailed(productId, error)))
                self.refreshReceipt()
            }
        }
    }
    
    private func refreshReceipt() {
        SwiftyStoreKit.fetchReceipt(forceRefresh: true) { result in
            switch result {
            case .success(let receiptData):
                let encryptedReceipt: String = receiptData.base64EncodedString(options: [])
                printd("[IAP] Fetch receipt success:\n\(encryptedReceipt)")
            case .error(let error):
                printd("[IAP] Fetch receipt failed: \(error.localizedDescription)")
                printd(loggingLevel: .error(error))
                printd(loggingLevel: .error(IAPPurchaseServiceError.refreshReceiptError(error)))
            }
        }
    }
    
}

```

### Verify Subscriptions

```swift
//
//  IAPVerifySubscriptionsService.swift
//  DailyWeightMeasure
//
//  Created by paige shin on 2022/09/14.
//

import Foundation
import Combine
import StoreKit
import SwiftyStoreKit

// This will contain the state of each items
// You can check this value to check if each one of items is stil valid
// Why creating this object?
// When entering the app, this persisted data will help determine if user is still within valid period of subscription
// And you can update subscription status to the server
struct IAPItemResult: Hashable, Equatable {
    let productId: String
    var expiryDate: Date?
    var receiptItems: [ReceiptItem]
    let subscribed: Bool
    
    func hash(into hasher: inout Hasher) {
        hasher.combine(productId)
    }
    
    static func ==(lhs: Self, rhs: Self) -> Bool {
        return lhs.productId == rhs.productId
    }
}

enum IAPVerifySubscriptionsServiceError: Error {
    case icloudNotConnected
}

extension IAPVerifySubscriptionsServiceError: LocalizedError {
    var errorDescription: String? {
        switch self {
        case .icloudNotConnected:
            return "icloud is not connected"
        }
    }
}

protocol IAPVerifySubscriptionsServiceProtocol {
    var iapItemResults: CurrentValueSubject<Set<IAPItemResult>, Never> { get }
    var processed: Bool { get }
    init(networkConnectivity: NetworkConnectivityProtocol,
         icloudConnectivity: ICloundConnectivityProtocol,
         productIds: Set<String>,
         sharedSecret: String)
    func verifySubscriptions() throws
    func clearIAPItemResults()
}

struct IAPVerifySubscriptionsService: IAPVerifySubscriptionsServiceProtocol {
    
    private(set) var iapItemResults: CurrentValueSubject<Set<IAPItemResult>, Never> = CurrentValueSubject([])
    var processed: Bool {
        get {
            return self.iapItemResults.value.count == self.productIds.count
        }
    }
    
    private var networkConnectivity: NetworkConnectivityProtocol
    private var icloudConnectivity: ICloundConnectivityProtocol
    private var productIds: Set<String>
    private var sharedSecret: String
    
    init(networkConnectivity: NetworkConnectivityProtocol,
         icloudConnectivity: ICloundConnectivityProtocol,
         productIds: Set<String>,
         sharedSecret: String) {
        self.networkConnectivity = networkConnectivity
        self.icloudConnectivity = icloudConnectivity
        self.productIds = productIds
        self.sharedSecret = sharedSecret
    }
    
    func verifySubscriptions() throws {
        if !self.icloudConnectivity.isICloudContainerAvailable() {
            printd(loggingLevel: .error(IAPVerifySubscriptionsServiceError.icloudNotConnected))
            throw IAPVerifySubscriptionsServiceError.icloudNotConnected
        }
        self.productIds.forEach {
            self.verifySubscription(productId: $0) { iapItemResult in
                self.updateIAPItemResults(iapItemResult)
            }
        }
    }
    
    private func updateIAPItemResults(_ itemResult: IAPItemResult) {
        if self.iapItemResults.value.contains(itemResult) {
            self.iapItemResults.value.update(with: itemResult)
        } else {
            self.iapItemResults.value.insert(itemResult)
        }
    }
    
    private func verifySubscription(productId: String, completion: @escaping(IAPItemResult) -> Void) {
        if !networkConnectivity.isConnectedToNetwork() {
            completion(IAPItemResult(productId: productId, expiryDate: nil, receiptItems: [], subscribed: false))
            return
        }
        
        let appleValidator: AppleReceiptValidator
#if DEBUG
        appleValidator = AppleReceiptValidator(service: .sandbox, sharedSecret: self.sharedSecret)
#else
        appleValidator = AppleReceiptValidator(service: .production, sharedSecret: self.sharedSecret)
#endif
        SwiftyStoreKit.verifyReceipt(using: appleValidator) { result in
            switch result {
            case .success(let receipt):
                // Verify the purchase of a Subscription
                let purchaseResult = SwiftyStoreKit.verifySubscription(
                    ofType: .autoRenewable, // or .nonRenewing (see below)
                    productId: productId,
                    inReceipt: receipt)
                switch purchaseResult {
                case .purchased(let expiryDate, let items):
                    printd("Success Cased for IAPVerifySubscriptionsService!")
                    printd("Purchased Products Expiry Date: \(expiryDate)")
                    completion(IAPItemResult(productId: productId, expiryDate: expiryDate, receiptItems: items, subscribed: true))
                case .expired(let expiryDate, let items):
                    completion(IAPItemResult(productId: productId, expiryDate: expiryDate, receiptItems: items, subscribed: false))
                case .notPurchased:
                    completion(IAPItemResult(productId: productId, expiryDate: nil, receiptItems: [], subscribed: false))
                }
            case .error(_):
                // ERROR
                completion(IAPItemResult(productId: productId, expiryDate: nil, receiptItems: [], subscribed: false))
                self.refreshReceipt()
            }
        }
        
    }
    
    
    private func refreshReceipt() {
        SwiftyStoreKit.fetchReceipt(forceRefresh: true) { result in
            switch result {
            case .success(let receiptData):
                let encryptedReceipt: String = receiptData.base64EncodedString(options: [])
                printd("[IAP] Fetch receipt success:\n\(encryptedReceipt)")
            case .error(let error):
                printd("[IAP] Fetch receipt failed: \(error.localizedDescription)")
            }
        }
    }
    
    func clearIAPItemResults() {
        self.iapItemResults.value.removeAll()
    }
    
}

```

### Restore Service

```swift
//
//  IAPRestoreService.swift
//  DailyWeightMeasure
//
//  Created by paige shin on 2022/09/14.
//

import Foundation
import Combine
import StoreKit
import SwiftyStoreKit

enum IAPRestoreServiceError: Error {
    case networkNotConnected
    case restoreFailedForSome([(SKError, String?)])
    case nothingToRestore
}

extension IAPRestoreServiceError: LocalizedError {
    var errorDescription: String? {
        switch self {
        case .networkNotConnected:
            return "Network is not connected"
        case .restoreFailedForSome(let items):
            var messages: String = ""
            for item in items {
                if let message: String = item.1 {
                    messages += "`\(item.0.localizedDescription)` with message `\(message)`"
                } else {
                    messages += "`\(item.0.localizedDescription)`"
                }
            }
            return messages
        case .nothingToRestore:
            return "Nothing to restore..."
        }
    }
}

protocol IAPRestoreServiceProtocol {
    init(networkConnectivity: NetworkConnectivityProtocol, sharedSecret: String)
    func restore() -> AnyPublisher<[Purchase], IAPRestoreServiceError>
}

struct IAPRestoreService: IAPRestoreServiceProtocol {
    
    private var networkConnectivity: NetworkConnectivityProtocol
    private var sharedSecret: String
    
    init(networkConnectivity: NetworkConnectivityProtocol, sharedSecret: String) {
        self.networkConnectivity = networkConnectivity
        self.sharedSecret = sharedSecret
    }
    
    func restore() -> AnyPublisher<[Purchase], IAPRestoreServiceError> {
        Deferred {
            Future { promise in
                if !self.networkConnectivity.isConnectedToNetwork() {
                    printd(loggingLevel: .error(IAPRestoreServiceError.networkNotConnected))
                    promise(.failure(.networkNotConnected))
                    return
                }
                
                SwiftyStoreKit.restorePurchases(atomically: false) { results in
                    
                    results.restoredPurchases.forEach { product in
                        if product.needsFinishTransaction {
                            SwiftyStoreKit.finishTransaction(product.transaction)
                        }
                    }
                    
                    if results.restoreFailedPurchases.count > 0 {
                        printd("Restore Failed Purchases")
                        printd("Total Number Of Restore Failed Purchased: \(results.restoreFailedPurchases.count)")
                        printd(loggingLevel: .error(IAPRestoreServiceError.restoreFailedForSome(results.restoreFailedPurchases)))
                        promise(.failure(.restoreFailedForSome(results.restoreFailedPurchases)))
                        return
                    }

                    if results.restoredPurchases.count > 0 {
                        printd("Restored Purchases")
                        printd("Total Number Of Restored Purchased: \(results.restoredPurchases.count)")
                        promise(.success(results.restoredPurchases))
                        return
                    }

                    printd(loggingLevel: .error(IAPRestoreServiceError.nothingToRestore))
                    promise(.failure(.nothingToRestore))
                }
            }
        }
        .eraseToAnyPublisher()
    }
}



```
